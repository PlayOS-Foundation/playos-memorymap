# 09 — Next Steps & How to Resume

> Last updated: 2026-09-22

## Where we are

Sprints 0–15 are complete and verified (14.5 has two parked verification checks;
15 has one — the desktop windowed run). Sprint 15's SDK ships the musl toolchain,
`libplayos`/`libraylib`, CMake/pkg-config, the `desktop` shim/profile, the QEMU
`emulator` profile, and the `sdk-reference` sample validated on `device` +
`emulator`. See [`05-sprint-status.md`](05-sprint-status.md) for head SHAs and
evidence.

## Next up: Sprint 16 — `playos-net` (Wi-Fi)

Sprint 16 is **reviewed and realigned but not started** (2026-09-22). It brings
Wi-Fi up with a D-Bus-free stack — `wpa_supplicant` + `dhcpcd` + a trusted
`playos-net` bridge — over the existing `control.sock`. Read
`playos-spec/src/sprints/Sprint-16.md` (its Realignment notes table lists every
correction) and `playos-spec/src/adr/ADR-0012-wifi-stack.md` before starting.
Ground truth: `board/ally/linux.config` has `# CONFIG_WIRELESS is not set` (T1 is
real work); `dhcpcd` is already enabled; there is no `playos-net` repo yet (start
it in `playos-refdistro/src/playos-net/`). T8's on-device Wi-Fi checks are
**hardware-gated** on the Ally's MT7921e; host/QEMU covers T1–T7 and the QEMU
half of T8.

### Parked checks to pick up when hardware/display allows

- **S15 desktop windowed run** — `sdk/scripts/build-desktop.sh
  playos-samples/sdk-reference`, then run `build-desktop/bin/game` in a window
  (full repro and required evidence in `Sprint-15.md` → Parked verification).
- S14.5's two installer checks and the S14 residuals are listed below.

Current pins (`versions.lock`, 2026-09-22): init `4f9c599`, compositor `45cbeb0`,
shell `f32727f`, spec `2a152b3`, samples `d662b5f`, runtime `4b6426a`,
platform-api `ff6ec10`. shell still lags its repo HEAD because those S15 commits
are host/SDK-side; re-check before a release.

S16 firmware note: the Ally's **internal** Wi-Fi is the AMD RZ616 = **MediaTek
MT7922** (`mt7921e`, PCI `14c3:0616`), so T1 enables
`BR2_PACKAGE_LINUX_FIRMWARE_MEDIATEK_MT7922` only; `_MT7921` is a sibling chip and
`_MT7922_BT` is Bluetooth-only (out of scope).

## Sprint 14 residuals (documented, not tasks)

1. **P1 — boot is 6.49 s to shell-ready, target < 5 s.** Attributed with
   `playos-refdistro/scripts/boot-timeline.sh`. The Ally has **no bootloader**
   (EFI stub boot; the command line is compiled in), so nothing can pre-declare
   the active slot. The kernel log shows the NVMe partitions exist at 1.76 s and
   the kernel hands over at 1.81 s, while init's first ESP mount is at 4.61 s, so
   **~2.7 s is spent inside the first (initramfs) init** — unlogged, because
   `/data` is not mounted yet. That phase is the actual target. What remains:
   - **RESOLVED 2026-09-20: the installed boot is 3.28 s (was 10.55 s), under the
     5 s target.** Each image now declares itself on its compiled-in command line
     (`playos.live=1` / `playos.installed=1`) instead of init trying to detect the
     boot medium - which is impossible here (the stick does not exist at the pivot
     decision; the firmware reports the same invalid `BootCurrent` for a stick
     boot and an installed boot). `scripts/build-install-kernel.sh` builds both
     kernels and the installer writes the installed one to the target ESP. Live
     path number still to confirm (its log is on the stick).
   - **Ship a fresh kernel + minimal initramfs as the install payload.** Now
     optional rather than critical: it would shave the ~1.8 s kernel hand-off
     (which unpacks the 198 MB live rootfs an installed system never uses) and
     shrink the payload. The boot marks showed the first init
     is an *older binary*: it runs from the kernel's embedded initramfs, which
     lives on the ESP and is never touched by A/B payloads (they only write the
     inactive slot's `rootfs.squashfs`). Its 2.6 s phase is therefore both
     unfixable and unmeasurable from the rootfs side. Booting the freshly built
     USB image live is the test; a reinstall propagates it. Doing it minimally
     also drops the ~1 s of unpacking a live rootfs an installed system never
     uses.
   - **Make the first init visible (`/dev/kmsg` markers), then cut what it
     shows.** Candidates: the ESP FAT mount + sync, the squashfs
     mount, the recovery button check, boot-stage FAT writes. Measured 0.10 s
     warm for `udevadm trigger`+`settle`, so that is not automatically guilty.
   - **Ship a minimal-initramfs kernel for the installed path (secondary).**
     `CONFIG_INITRAMFS_SOURCE=rootfs.cpio` embeds a 198 MB live rootfs; the
     1.81 s kernel hand-off bounds the unpack at ~1 s. The installer already
     writes `/BOOTX64.EFI` to the target ESP, so this is a payload change plus a
     second kernel build.
   - **Parallelise the shell's GL-context creation** with the compositor
     configure wait (~0.9–1.4 s, and it varies — measure a few boots first).
2. **F3 — no-DRM-device case.** Recovery needs no GL now (`playos-recovery`), but
   a machine with *no DRM device at all* (no compositor, not even software)
   would still need a kernel-console text UI.

## Open follow-ups

1. **Installer `wipefs` follow-up (from 11.5).** Historically `wipefs` did not
   reliably clear the inactive slot on reinstall (fresh installs pivoted fine
   regardless). The installer now releases the target's mounts itself
   (`playos_format_release_target`) and captures child output, which was written
   with this in mind — but the symptom has not been re-tested since. Worth one
   reinstall to confirm or close.
2. **Installer progress ownership — now tracked as
   [Sprint 14.5 Shell-Owned Install Progress](../playos-spec/src/sprints/Sprint-14.5.md)**
   (recorded 2026-09-13). Today the shell owns the front-end (disk list, hold-A
   confirm) and the standalone installer takes the screen for the destructive
   phase, styled to match. S14.5 keeps one engine (`libplayos-install`) and adds
   a supervised screen-less `playos-install-worker` with `PrepareInstall` /
   `InstallProgress` IPC, so the shell draws progress, completion and errors.
   Polish only — the current flow is verified end to end, and the standalone
   installer stays as the no-shell fallback.
3. **Samples are "non-cooperative":** they do not ack BACKGROUND within 500 ms,
   so init SIGSTOPs them when the overlay opens. Works, but they should ack like
   cooperative games.
4. **Internal install runs the old kernel.** The A/B payload is the rootfs, so
   the installed system has today's userspace but not the F3 kernel
   (SimpleDRM): a reinstall from the current USB image aligns them.
5. **`playos-memorymap` is pushed normally** (fixed 2026-09-20). The earlier
   "no pushable remote" note was wrong: the repo was reachable all along, but its
   remote was named `playos-memorymap` instead of `origin`, so `git push origin`
   failed with "No such remote" - which read like a 404. The remote is renamed,
   40 commits are pushed, and tracking is set. Safety bundle (refreshed):
   `~/playos-memorymap.bundle`.

6. **QEMU dev-rig mismatch:** the QEMU guest's `/data/log/compositor-stderr.log`
   did not show the newest compositor lines although `rootfs.cpio` contains the
   code. The device proves the code is fine; dev-tooling only, unresolved.

Documented-but-unfixed gaps from `playos-spec/src/testing.md`: **P5** hostname
identity (`uname -n` reports `(none)`), **P6** dev-image tools missing
(`evtest`, `modetest`, `weston-info`), **P7** `playos-ctl` specified but not
implemented (on-device diagnostics mean reading `/data/log/*` and `/sys`).

## Validation loop: overlay, screenshots, input

Build `make ally-dev-usb-image`, flash, then check the gestures, the pause
overlay and screenshots:

- **Gestures (swapped in S14, verified on hardware):** tap **COMMAND** =
  screenshot (in game and on the shell UI); tap **ARMOURY CRATE** = in-game menu.
  A tap/hold split on one button is impossible — hid-asus reports both as
  momentary pulses. Settings row: "Screenshot on COMMAND".
- **Screenshots:** `screencopy` captures the composited output to
  `/data/screenshots/playos-<epoch>.<ms>.png`; the wl_shm format must be honoured
  (`0x34324258` = XBGR8888 on the Ally) or red/blue swap. `playos-recovery`
  writes its own PNGs (`recovery-<epoch>.png`) because the shell's gesture is
  unavailable when the shell is the thing that failed.
- **Overlay:** hidden overlay ignores the gamepad, d-pad decoded from `ABS_HAT`,
  focus list (Resume / Quit / Profile), B resumes, Quit needs a held A, and the
  first input poll after `about_to_show` is discarded (it used to replay the
  gameplay backlog and dismiss the menu 2 ms after it appeared).
- **Idle behaviour:** the shell is damage-driven — idle it settles at ~8 fps and
  ~2.6% of a core. Only discrete input (keys, d-pad) counts as activity: the
  right stick rests with a ±128 `ABS_RY` oscillation (~65 events/s), so raw
  evdev traffic can never be the activity signal. Diagnostic screens (the Input
  tab's Live Input Test) ask for full rate explicitly.
- **Instrumentation to read first:** `playos-compositor: fps shell=N game=M
  (commits/s) present zero-copy=X copied=Y` — per-role frame rate and how frames
  reached the panel (zero-copy = direct scanout; measured 100% on the Ally).

## Resolved recently (do not re-investigate)

2026-09-12: F1 (`RollbackSlot` corrupted `boot.json`), F2 (4 s recovery watch on
every boot), F4 (hidden overlay consumed gamepad input; Ally d-pad not decoded),
B-in-game quitting, screencopy SIGABRT + role leak + red/blue swap, overlay
first-show instant dismiss.

2026-09-13: **F3 closed** (SimplEDRM + compositor software path + the GL-free
`playos-recovery` client, verified in QEMU and on the Ally); **P2/P3/P4** (in-game
fps, 100% direct scanout, damage-driven shell); **P1 first cuts** (7.66 → 6.49 s);
**T10 install verified end-to-end** (8/8 steps, seamless handoff, failure returns
to the shell).

## Suggested reading

- [`02-architecture.md`](02-architecture.md) — boot/IPC/storage
- [`04-build-and-test.md`](04-build-and-test.md) — commands
- [`07-security.md`](07-security.md) — what the sandbox does (so you can
  recognize success/failure in logs)
- [`08-gotchas.md`](08-gotchas.md) — avoid re-discovering known traps

## Signals that Sprint 12 sandbox is active (for log spelunking)

- `manifest signature verified: …` or `WARN: game manifest is UNSIGNED …`
  in the init log at spawn time.
- No `WARN: Landlock unsupported` / `WARN: Landlock setup failed` /
  `WARN: seccomp filter failed` lines at game launch.
- Game log shows the title running; `/data/saves/<id>` owned by uid 1001.
- `ps` shows the game under PID 1 as `playos-game`.

## Parked (revisit later, agreed 2026-09-22)

Two S14.5 verification checks, with exact repro steps in
`playos-spec/src/sprints/Sprint-14.5.md`:

1. **Forced failure** — kill `playos-install-worker` mid-install; expect the error
   card (not a stalled bar). The sprint's own acceptance line.
2. **Standalone installer rerun** — `PLAYOS_INSTALL_TARGET=… playos-installer`;
   confirms the engine extraction left that front-end unchanged.

Carried from Sprint 14, unchanged:

3. **Live-path boot 5.63 s** vs the <5 s target — 0.63 s over, device-bound (the
   dock's USB hub chain means the stick's partitions are not usable until ~4.6 s).
   The only fix is deferring the `/data` mount; the installed path is 3.28 s and
   meets the target.
4. **T9's CI release run** — `release.yml` exists and the signing is verified by
   hand on hardware; one green CI run remains.
5. **F3's no-DRM case** — recovery no longer needs GL, but a machine with no DRM
   device at all would still need a console text UI.
6. **Samples don't ack BACKGROUND** — init SIGSTOPs them instead; cosmetic.
7. **QEMU dev-rig log mismatch** — dev tooling only.

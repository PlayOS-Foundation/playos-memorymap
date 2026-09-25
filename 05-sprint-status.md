# 05 — Sprint Status

> **Last updated: 2026-09-22** — Sprints 11.5, 11.6, 12, 13, 13.6, 13.7, 14, and 14.5 are closed (validated on-device; 14.5 has two parked verification checks). **Sprint 15 (Game Developer SDK): done** — T1–T7 verified; T8 verified for `device` + `emulator` with one parked check (the desktop windowed run, no display in the session). **Sprint 16 (`playos-net`/Wi-Fi): not started — reviewed and realigned 2026-09-22** (ADR-0012; on-device acceptance is hardware-gated).
> Specs live in `playos-spec/src/sprints/`; this file summarizes state and evidence. Head SHAs are as of 2026-09-22.

## Head SHAs (all repos clean on `main`)

| Repo | HEAD |
|---|---|
| playos-spec | `3f9c580` spec: S16 T3+T4 done (playos-net verified on hardware) |
| playos-init | `7d2d9f9` ipc: network message types (S16-T4) |
| playos-compositor | `45cbeb0` compositor: report direct scanout (S14 P3) |
| playos-runtime | `4b6426a` trusted: StartInstaller carries the payload device (S14.5) |
| playos-refdistro | `17fac1a` versions.lock: bump init (network IPC types) + spec (S16 T3+T4) |
| playos-platform-api | `ff6ec10` input(evdev): stop closing device nodes during discovery |
| playos-shell | `c055e26` audio: hand the playback PCM to a foreground game (ADR-0007) |
| playos-samples | `d662b5f` invaders: procedural SFX (no asset files) |
| playos-raylib | `dbc56a8` (6.0 tag, pinned in versions.lock) |
| playos-tools | `ce8f1e9` sdk: implement the emulator profile (S15-T7) |
| others | unchanged (docs/cloud) |

## Sprints 0–11: complete

| Sprint | Deliverable | Validation |
|---|---|---|
| 0 | Buildroot + UEFI QEMU boot | QEMU |
| 1 | playos-init PID 1 | QEMU |
| 2 | wlroots compositor skeleton | nested/headless |
| 2.5 | IPC unification audit | QEMU + spec update |
| 3 | ROG Ally kernel bring-up, input contract | device |
| 4 | Native DRM/KMS compositor + test client | device 119.8 fps |
| 5 | Raylib shell + trusted control + crash restart | device |
| 5.5 | Raylib 6.0 shell migration | device |
| 5.6 | Repo extraction + component pins (versions.lock) | build |
| 6 | Persistent storage + game discovery + seeding | device |
| 7 | Game launch/lifecycle/overlay/SYSTEM button | device |
| 8 | ALSA audio (miniaudio→ALSA, card routing) | device |
| 9 | Power/battery/thermal/suspend | device |
| 9.5 | Brightness control | device |
| 10 | Installer + NVMe deploy + factory reset | device (Ally) |
| 11 | A/B updates: boot.json slots, .playosb sign/verify, ApplyUpdate IPC, shell update UI | QEMU + device |

## Sprint 11.5 — pivot-to-squashfs boot + A/B validation

**Complete (validated on-device 2026-08-23).** T1–T4 (bootable squashfs rootfs;
`playos_pivot_to_active_slot()` switch_root idiom; active-slot boot.json wiring;
3-strike rollback) plus T5's 6-case A/B matrix all verified on the ROG Ally.
One non-blocking follow-up remains tracked: the installer's `wipefs` doesn't
reliably clear the inactive slot on reinstall (cosmetic — a fresh install still
pivots correctly).

## Sprint 11.6 — Dropbear SSH + wired USB-C Ethernet

**Complete (validated on-device 2026-08-23).** QEMU and the ROG Ally both verified
DHCP lease + pubkey login over USB-C Ethernet. Installer seeds
`/data/ssh/authorized_keys` from the installer USB into the target NVMe
`playos-data` during format.

## Sprint 12 — Security Hardening (implemented 2026-08-22, closed on-device)

| Task | State | Evidence |
|---|---|---|
| T1 game credential drop (uid 1001, NO_NEW_PRIVS, cap drop) | **done** | `playos-init/src/security/sandbox.c`; hard-fails launch on drop failure |
| T2 Landlock default-deny allowlist | **done** | `src/security/landlock.c`; host test passes (allowlist + default-deny) |
| T3 seccomp deny-list + prctl(PR_SET_SECCOMP) block | **done** | `src/security/seccomp_filter.c`; host test skips (sandboxed build host); exercised on-device |
| T4 DRM primary-node denial | **done** | Landlock default-deny on `/dev/dri` + no `drm` group |
| T5 reserved-button isolation (compositor) | **done** | `playos-compositor/src/system_button.c` intercepts SYSTEM + QUICK_MENU keycodes; builds clean |
| T6 control.sock `0660 root:playos-trusted` + peer check | **done** | already enforced; policy test in `playos-runtime/tests/test_trusted_control.c` |
| T7 production image strip + lint | **done** | `playos_ally_production_defconfig`, `board/ally/post-build.sh`, dev overlay split, `make ally-production-build`; `production-build.yml` CI asserts no debug artifacts + QEMU `--production` |
| T8 Ed25519 manifest verify (warn-only) | **done** | self-contained Ed25519 (RFC 8032 + Python cross-verify), `scripts/sign-manifest.sh` |
| T9 Secure Boot docs + dev keys | **done** | `keys/dev/*`, `scripts/sign-efi.sh`, `security-model.md` updated; CI wiring in `production-build.yml` |

**Closed on-device 2026-08-23.** The Ally kernel `CONFIG_SECURITY` gap was fixed
(`CONFIG_SECURITY=y`, `CONFIG_SECURITY_LANDLOCK=y`), the 11/11 on-device self-test
passed, and games launch/exit clean (`playos-shell dde9f57` B-exit fix). Games are
now sandboxed with supplementary groups `audio`, `render`, `input`.

## Sprint 13 (Intel expansion) — on-device boot confirmed, closing out

Target pinned to **ASUS ZenBook UX530** (Kaby Lake, HD 620, NVIDIA dGPU ignored).
Host-testable work is done and committed: GPU-selection scoring extracted to
`gpu_score.c` with ctest `gpu-select` (compositor `195664c`); vendor-agnostic
hwmon matchers in platform-api (`d9d5bae`) and init (`41c071b`); Intel defconfig
`playos_intel_pc_defconfig` (i915 + Mesa iris/LLVM + HDA), `make intel-*` targets,
`gen-intel-usb-image.sh`, and `intel-build.yml` CI (refdistro `9b29a32`, RAPL/POWERCAP
fix `90683cc`); `hardware-matrix.md` + `backend-portability.md` docs (spec `b937a95`).

**On-device (2026-08-26, LiveUSB `playos-intel-usb.img`):** boots to the shell;
game library renders; sample game launched + rendered; USB gamepad input works;
battery % + thermal state render in the status bar. Remaining: `audio-sine`,
system-button/lifecycle, and SSH log capture for the explicit `0x8086`/Mesa-Iris
strings (no USB-C Ethernet adapter yet). AMD regression re-run still pending.
Evidence tracked in `playos-spec/src/sprints/Sprint-13.md` (spec `e874c13`,
pin bumped `be600a0`).

**Closed 2026-08-30.** S13.6 (shared SDL_GameControllerDB mapping DB + Sony
DualSense/DualShock preferred names) landed in platform-api `a217562`, shell
`f481b2b`, and spec `731d7b6`. Also this session: init `8afc134` hardens the
installer by refusing a `playos-data` on an internal/install-target disk (sysfs
`removable` flag replaces the `nvme`/`mmcblk` heuristic), and refdistro `65e4102`
syncs before reboot + bumps the init pin.

**Closed 2026-09-01.** S13.7 (Live-USB / Installer Image Consolidation) is done
end-to-end: runtime installer handoff (init `5bd4575`…`53fc75a`), shell Settings
install action (df6ea9d…fb7b978), refdistro image consolidation (18d95a3…),
and T7 validation — QEMU headless runtime-install **PASSED** plus on-device ROG
Ally: live USB boots without slot pivot (ESP live-USB marker), Settings install
works, reboot into installed NVMe, dev SSH key auto-seeded and SSH reachable.
Spec `Sprint-13.7.md` T7 done (spec `2dd083e`, pin updated in refdistro).

**Dev release `dev-v0.3.0` (2026-09-01).** Pushed tag `dev-v0.3.0` →
`PlayOS Dev Images Release` workflow builds and publishes
`playos-ally-dev-usb.img.xz`, `playos-intel-dev-usb.img.xz`, and
`playos-ally-prod-usb.img.xz` + SHA256 as a GitHub pre-release.
CI notes: workflows now run on `ubuntu-24.04` (host-mesa3d needs GCC > 22.04),
`actions/cache` covers Buildroot `dl/` + per-board host toolchains, and the Intel
kernel check forbids GPU drivers only (`X86_AMD_PSTATE` is a CPUFreq driver forced
by Kconfig `select` and is intentionally not gated).
Sprint 14 in progress; S14-T10 adds "installer as a PlayOS app with console-free
seamless handoff" (spec `b68509a`).

**Sprint 14 implementation (2026-09-12).** T1/T2 API freeze done;
T3 Doxygen docs + examples + getting-started (`f3e629c`) done;
T4 tag-triggered release pipeline done (`dev-v0.3.0` published);
T5 MVP smoke checklist + `scripts/mvp-smoke.sh` committed (`24dd9c8` +
`c6fc07e`) — on-device 19-criterion run pending; T6 recovery core with
button-hold entry (`1729329`, `e3c0091`) done — later closed by F3;
T7 perf baseline checklist + collector (`63a4d67` + `59e3fd4`) — measurements
pending; T8 spec docs + ADRs (`182b138`) done; T9 production defconfig +
sign scripts + release lint done — final signed v0.3.0 run + SDK-compile
verification pending; T10 console-free installer handoff + splash
(`49fd28e`, `1a7ace4`) done — shell app-style installer entry pending.
Also landed: SYSTEM/COMMAND → pause overlay via ShowOverlay/HideOverlay IPC
(runtime `7d65fe3`, init `0c4fa77`, compositor `0404ffd`, shell `6e9211c`).
Pins bumped in refdistro `bce8126`. Remaining: T9/T10 shell front-end
polish, on-device smoke + perf runs, SimpleDRM recovery validation.

**Recovery fixes (2026-09-12).** The recovery menu's Rollback no longer rewrites
`/EFI/playos/boot.json` — that shell edit wrote one byte too many over the
value's comma, corrupting the file so init could not parse it and the rollback
silently no-op'ed on the next boot. The shell now sends a `RollbackSlot` IPC
and init applies the real `boot_slot_rollback()` semantics (current slot `bad`,
target `pending`, boot count reset) before rebooting. The late recovery button
watch is now polled non-blocking from the supervision loop, so a normal boot is
no longer delayed 4s and no longer prints a recovery prompt to the console.
init `0094e83`, runtime `01c193b`, shell `1bc7403`, spec `f5f1417`, pins
`c341f8f`. Still open: on-device verification of the fixes (F4 below — the
overlay client processes gamepad input even while hidden, so B may quit the
game during play; needs an on-device check).

**Overlay input fixes (2026-09-12, from on-device testing).** F4 confirmed on
hardware: the hidden overlay still acted on the gamepad, so B during play quit
the game. Fixed by discarding decoded input while `!st.visible` and by making
the compositor send `about_to_hide` whenever it clears overlay visibility
(refdistro overlay `95747fc`, compositor `6fd8f63`). Also fixed the d-pad: the
overlay decoded only `BTN_DPAD_*`, but the ROG Ally reports the d-pad as
`ABS_HAT0X/ABS_HAT0Y` (xpad/hid-asus), so d-pad volume, profile change, and the
power-menu cursor did nothing — both forms are now decoded. Overlay sub-menu
mode resets on hide. Spec reconciled in `playos-overlay-spec.md` (`b38c1e5`).

**Pause menu redesign (2026-09-12, requested on-device).** Owner wants B to mean
resume and Quit to be a menu entry. The quick menu is now a focus list — Resume
Game / Quit Game / Performance Profile: D-pad Up/Down moves focus, A activates,
B resumes (never quits), D-pad Left/Right steps volume, SELECT opens the power
menu. Quit Game is item-only and requires holding A ~0.9 s (progress readout);
`poll_input` now tracks A's held state for that. Profile opens from the menu
item instead of the old d-pad L/R modifier; volume moved from Up/Down to
Left/Right. refdistro `8424797`, spec `f888d29`.

**Full-output screenshots + IPC latency (2026-09-12).** Two on-device
complaints: screenshots could never show the game, and COMMAND felt slow
in-game. Root causes/fixes:

- Screenshots read the shell's own framebuffer (`LoadImageFromScreen`) and
  never ran while a game was foreground. The compositor now exposes
  wlroots' `zwlr_screencopy_manager_v1` (`f20597b`) and the shell captures
  the composited output from it (`src/screencopy.c`, shell `3f72c2f`),
  converting the shm buffer to RGBA and writing the PNG with `ExportImage`;
  the surface grab remains as a fallback. Works in-game and with the pause
  overlay up.
- COMMAND gesture: in-game a tap opens the overlay and a hold >= 700 ms takes
  a screenshot; on shell screens a press takes one. Captures run in the input
  section so they work while suspended. The Settings toggle now persists to
  `/data/config/screenshot` and defaults to on.
- The in-game COMMAND path (shell -> init -> compositor) waited up to ~1 s:
  init's loop polled its sockets non-blocking then `nanosleep(1s)`. It now
  waits on the fds with `poll()` (init `3c7309d`); the 1 Hz housekeeping and
  the one-shot ticks (late audio, mark-good) are time-based so early wakeups
  do not fast-forward them.
- Known gap: in-game captures have no on-screen feedback (the shell surface
  is hidden); only a log line. Also `playos-overlay-spec.md` still describes
  aspirational UI not in the implementation.

**Gesture correction (same day, after on-device test).** The hold-COMMAND
gesture never worked: a raw evdev trace showed the Ally's Command Center button
emits `KEY_F16` press **and** release inside one poll (~1 ms apart) — a
momentary pulse with no sustained state, so `shell_input_button_held()` could
never see it. That also regressed the in-game COMMAND tap → pause overlay,
which had used the edge query. Fixed in shell `ec0f9e5`: gestures are now
edge-triggered and **ARMOURY CRATE (`SYSTEM`, reserved and otherwise unbound)
takes the screenshot anywhere**, while COMMAND keeps the pause overlay in game
and the screenshot on the shell UI. Added an explicit
`screenshot requested (…)` log line and a 0.4 s debounce. See
[`08-gotchas.md`](08-gotchas.md) for the input lessons.

**B during gameplay: two separate causes.** "B returns me to the shell
in game" looked like a platform bug but was the samples' own code: all 11 games
exited on B (`IsGamepadButtonPressed(… RIGHT_FACE_RIGHT)` → `break`, 8 of them
printing `B = QUIT` on screen) — a leftover from before the pause overlay
existed. Removing it matters because a reference distribution's samples teach
UX: quitting is now the overlay's Quit Game (→ `PLAYOS_LIFECYCLE_TERMINATE`)
only. The PlayOS side was already correct — the hidden overlay discards its
input, and B is not a reserved key, so a third-party game may still use B
however it likes. Note the samples never relied on `WindowShouldClose()` (the
PlayOS backend does not feed it), so the lifecycle is genuinely their only exit.

**S14 progress (2026-09-12).** T5/T7/T9/T10 all moved: the MVP smoke ran on hardware
(**18/19**, criterion 19 blocked by F3 — reports in `playos-refdistro/docs/`), the perf
baseline was measured (shell→game 1.00 s, SYSTEM→overlay 4–5 ms, exit→shell 28 ms pass;
boot→shell 7.16 s vs 5 s target; in-game FPS and direct scanout unverified) with gaps
P1–P7 filed in `playos-spec/src/testing.md`, the SDK headers were verified to compile,
link and run a minimal game on the host, and the installer became an app: the shell now
owns the disk picker + hold-A confirm (`SCREEN_INSTALLER`) and passes the chosen disk
through `StartInstaller` → `PLAYOS_INSTALL_TARGET`.

**F3 closed properly (2026-09-13): the recovery UI no longer needs GL.** Compositor-side software
rendering was not enough — the shell is a GL client and cannot initialise EGL against a pixman
compositor on the Ally, which produced an empty (blue) screen. The new `playos-recovery` package
is a `wl_shm` client that paints its own text (stb_truetype) and reads evdev directly; init starts
it when the shell cannot run in recovery, and the GL shell remains the preferred UI when it works.
Verified in QEMU with the `playos.noshell` hook reproducing the shell-failure case
(`docs/evidence/f3-recovery-client-no-gl-2026-09-13.png`). Not covered: a machine with no DRM
device at all (no compositor → would need a kernel-console UI).

**Verified on the Ally the same day** through the *automatic* path: recovery shell killed until
init's restart limit tripped → `recovery client launched (PID 505)` → the client
`registered as the shell role` (compositor: `trusted: shell client registered`) → menu rendered with
d-pad, log list/viewer working, and its own screenshot saved to
`/data/screenshots/recovery-<epoch>.png`. Evidence pair:
`f3-recovery-shell-menu-on-ally-2026-09-13.png` (shell's menu, status bar present) vs
`f3-recovery-client-on-ally-2026-09-13.png` (client's menu, no status bar).

**Earlier F3 work — compositor software path (2026-09-13).** Recovery no longer depends on
the accelerated GPU: the kernel provides SimplEDRM (`FB`/`SYSFB`/`SYSFB_SIMPLEFB`/`DRM_SIMPLEDRM`),
the compositor has a software (pixman) path that probes for a usable DRM device before creating
the backend once, and init forces `PLAYOS_RENDERER=pixman` for recovery plus restarts a dead
compositor in software mode. Along the way `playos.recovery` on the cmdline was found to be a
no-op (read before `/proc` was mounted). Verified with the new
`scripts/qemu-recovery-check.sh`: guest with **no GPU driver** (QEMU cirrus VGA → SimplEDRM
only) renders the recovery menu on screen — evidence
`playos-refdistro/docs/evidence/f3-recovery-menu-no-gpu-2026-09-13.png`, report
`docs/f3-recovery-software-rendering-2026-09-13.md`.
**Perf gaps P2 + P4 done and measured on the Ally (2026-09-13).** P2: the compositor logs each toplevel's commit rate per second (`fps shell=N game=M`) — measured compositor-side so it covers non-cooperative games; a sample game peaked at **120 commits/s**. P4: the shell is damage-driven — idle **55.5 → 8.0 fps** and shell CPU **7.4% → 2.6-2.8% of one core**; only discrete input counts as activity (the right stick rests with a ±128 `ABS_RY` oscillation, ~65 events/s, which pinned the old rule), analog-motion screens such as the Live Input Test ask for full rate explicitly, and the idle loop sleeps 4 ms because raylib's pacing lives inside `EndDrawing()`. Evidence in `playos-refdistro/docs/perf-baseline-report-2026-09-12.md` (follow-up section) and `08-gotchas.md`.

**P1 first cuts landed and measured (2026-09-13).** Boot attribution
(`scripts/boot-timeline.sh`) showed 4.51 s before init, 0.79 s to mount the ESP, a fixed 500 ms "grace
period" before spawning the shell, 0.93 s of shell EGL/window init and 0.44 s to the first frame. The
grace period is now a Wayland-socket connect-probe (logged: "ready after 0 ms") and the ESP retry polls
at 25 ms instead of backing off: **cold boot → shell ready 7.66 s → 6.49 s (−1.17 s)**, `system ready`
at 5.58 s. The ESP stage itself did not shrink — that wait is the kernel bringing up the NVMe, and it is
critical-path by construction: the Ally has **no bootloader** (EFI stub boot, command line compiled in),
so nothing can pre-declare the active slot — init must mount `/EFI`, read `boot.json` and pivot before
user-space starts, which cannot happen before the kernel exposes the NVMe. The real P1 item is elsewhere:
the installed system boots a kernel with the **live rootfs embedded** (`CONFIG_INITRAMFS_SOURCE=rootfs.cpio`,
198 MB) and then pivots to the squashfs, so every installed boot unpacks a payload only the live USB needs.

**P3 closed (2026-09-13): direct scanout is now measured, not assumed.** The compositor logs how each
frame reached the panel (`present zero-copy=N copied=M`; zero-copy = presented without a renderer copy).
On the Ally: a 30 s game run gave **1684 zero-copy / 0 copied = 100% direct** (`fps shell=0 game=56`),
and the whole session 2535 / 0. The same data shows the shell committing **0 frames/s while a game is
foreground**, i.e. the P4 gate holds in-game too.

**Sprint 14 closed 2026-09-13: T1-T10 all `done`** (T5 19/19 with criterion 19 met via F3; T7 with P2/P3/P4
measured and P1 improved and its residual documented). Residual items, none of them sprint tasks:
P1's remaining ~1.5 s to the 5 s boot target (a minimal-initramfs kernel for the installed path — the
big one; the ~0.6 s NVMe/ESP wait is hardware bring-up and is not removable; shell GL-init
parallelisation); F3's no-DRM-device case (no compositor at all needs a kernel-console UI); the
`testing.md` gaps P5 (hostname identity), P6 (dev-image tools), P7 (`playos-ctl` unimplemented); the
samples' missing BACKGROUND ack (they are SIGSTOPed when the overlay opens). Housekeeping: this repo has
renamed to `origin` and pushed on 2026-09-20 (the earlier "no pushable remote" note was a misread of `git push origin` against a remote named `playos-memorymap`), and the internal
install runs the new userspace but the old kernel (no SimplEDRM) until it is reinstalled from the USB image.

Remaining in Sprint 14: the rest of P1 only (the installed-path minimal initramfs — the ~2.5-3 s before
init is mostly unpacking the embedded live rootfs; the ~0.6 s NVMe/ESP wait is hardware; shell startup
~0.9-1.4 s).

**S14-T9 verified on hardware (2026-09-13).** Final signed-artifact run: production image
hygiene confirmed (no `/bin/sh`, busybox, dropbear/sshd), EFI kernel signed (`sbverify` OK),
prod `update.playosb` built with HMAC + payload hash independently verified, SDK headers
compile a minimal game with `-Werror` and run. Then the **A/B update was applied to the
installed Ally**: slot B written byte-exactly (`sha256(slot B)` == shipped
`rootfs.squashfs`), `boot.json` flipped to `b`/0.3.0/healthy, and the system booted slot B.
Evidence: `playos-refdistro/docs/t9-signed-artifacts-2026-09-13.md` + `playos-0.3.0-SHA256SUMS.txt`.
**Rollback half verified too (2026-09-13).** From slot B: reboot holding Volume Up → recovery
menu → Rollback → `RollbackSlot` IPC → `active slot b -> a, rebooting`; after the reboot `/` is
`/dev/nvme0n1p2` (slot A), `boot.json` has `slot_a {0.1.0, good}` and `slot_b {0.3.0, bad}` (the
slot rolled back *from* is quarantined). Both directions of the A/B contract are now
hardware-verified: update forward and roll back safely.

Still open for T9: the CI release run (`.github/workflows/release.yml`), which reproduces this
artifact set with CI secrets.

**S14-T10 verified on hardware (2026-09-13).** The installer now works end to end from the
shell: Settings → System → Install PlayOS to internal disk → disk → hold A → the handoff is
seamless (no blink, no console, compositor and `/data` stay up) → all 8 steps complete →
reboot boots the installed system. Evidence: `installer.log` shows steps 0-7 ok
(including the SSH-key seed) and `installer: target nvme0n1 is free of mounts`; the installed
`/data` is fully populated (11 games, sessions log), `boot.json` is slot a / good with
boot_count 1, and the internal disk has the documented 5 partitions
(512 MiB ESP, 2x4 GiB slots, 64 MiB misc, remainder data).
Three handoff bugs were found and fixed on the way (NULL trusted-client destroy listener →
compositor SIGSEGV; ack-after-teardown deadlock; `nvme0n1` name parsing leaving the ESP
mounted so `mkfs.fat` refused — see 08-gotchas.md).

Still open on-device: SimpleDRM/low-graphics recovery (F3), the last T9 signing step,
and the perf gaps.

## Where each sprint's detail lives

`playos-spec/src/sprints/Sprint-<N>.md` — each has goal, decisions locked,
task breakdown with status grid, acceptance criteria, and handoff notes.

**S14 closed 2026-09-20 - P1 resolved on the installed path.** The boot medium is now declared per image
(`playos.live=1` in the live image's kernel, `playos.installed=1` in the payload the installer writes to
the target ESP) instead of being detected at runtime, which is impossible here: at the pivot decision the
USB stick does not exist yet (the dock's hub chain; a hot-plug is ~270 ms) and the firmware reports the
same invalid `BootCurrent` for a stick boot and an installed boot. Installed boot: **10.55 s → 3.28 s**
to ShellReady, under the 5 s target. Live path: 5.63 s (0.63 s over, device-bound). Also closed: the
live-boot boot accounting (live sessions no longer advance the installed slot's counters), and the
`wipefs` follow-up (a full reinstall succeeded). Residuals: live-path 0.63 s, T9's CI release run, F3's
no-DRM case, `playos-memorymap` having no remote.

**S14.5 (Shell-Owned Install Progress) — 4 of 5 tasks done, 2026-09-22.** T1–T4 are
complete and the shell-driven install is verified on hardware end to end: the shell
stays on the installer screen, progress and percentage are visible, the success
card appears, and "A: Reboot now" boots the installed system (`/dev/nvme0n1p2`,
`boot.json` `slot_a` good). Two T5 checks are parked with instructions in the
sprint doc: the forced-failure error card, and a standalone-installer rerun.
Evidence and the four hardware-only defects it found:
`playos-refdistro/docs/s14.5-install-verification-2026-09-22.md`.

## Sprint 15 (Game Developer SDK) — in progress, T1–T6 done (2026-09-22)

T1–T7 are done and verified; see `playos-spec/src/sprints/Sprint-15.md` for the
grid and evidence. The shipped SDK is a 409 MB relocatable
`x86_64-buildroot-linux-musl` toolchain plus `libplayos` (`PLAYOS_API_VERSION 1`)
and `libraylib` with the `PLATFORM_PLAYOS` backend, CMake toolchain + `pkg-config`
files, and the `desktop` host shim/profile. A sample built entirely through the
SDK runs on the Ally (`device`, musl), in a host window (`desktop`), and in the
QEMU emulator (`emulator`). `scripts/export-sdk.sh` (refdistro) populates
`playos-tools/sdk/`; the profile scripts (`build-device.sh`, `build-desktop.sh`,
`build-emulator.sh`) and `docs/sdk.md` live in `playos-tools`.

- **T1** toolchain tarball (relocatable, verified by relocating + compiling)
- **T2** `libplayos` headers + musl libs
- **T3** musl `libraylib` with `PLATFORM_PLAYOS`
- **T4** CMake toolchain + `pkg-config` for `device`
- **T5** desktop host shim (keyboard/gamepad → controller ABI, lifecycle no-ops)
- **T6** desktop build profile
- **T7** emulator profile — done. `playos.autostart=<game-id>` in init
  (`4f9c599`), `playos_emulator_defconfig` + `make emulator-build` +
  `scripts/emulator-run.sh` (refdistro `278aa6c`), SDK `build-emulator.sh`
  (tools `ce8f1e9`). Verified: the SDK-built bunnymark boots in the minimal
  QEMU image, is launched by init under the S12 sandbox, and the compositor
  logs `game surface added to scene (role 3)` + `fps shell=0 game=1`. Design +
  measured evidence: `playos-spec/src/sdk-emulator-profile.md`.
- **T8** reference sample across all profiles — done (1 parked check).
  `playos-samples/sdk-reference/` (`3d98516`) is one `main.c` built entirely
  through the SDK for all three profiles, exercising system/lifecycle/input/
  storage/logging over an animated raylib scene. `device` builds musl;
  `desktop` builds glibc; the `emulator` run passes end to end (same evidence
  shape as T7, with the sample's own `sdk-reference 1.0.0 starting` /
  `saves at /data/saves/...` lines). The desktop **windowed run** is parked
  (no `DISPLAY`/`WAYLAND_DISPLAY` in the session); its repro and required
  evidence are in `playos-spec/src/sprints/Sprint-15.md` → Parked verification.
  Results: the sample's `README.md`.

  A second, fuller sample followed: **`playos-samples/invaders/`** (`586784d`) —
  a complete single-screen arcade shooter (marching fleet, shields, bombs,
  lives, levels, persisted high score) using only the public API + raylib. Built
  musl (device) and glibc (desktop) through the SDK; the emulator run launches it
  and the compositor claims `game surface added to scene (role 3)`. It **ships in
  the image**: `playos-samples.mk` installs it to
  `/usr/share/playos/games/com.playos.sample-invaders` (verified with
  `make playos-samples`), and the samples pin is bumped. Its emulator run could
  not produce the `game=N` commit-rate sample because this host has no readable
  `/dev/kvm` (TCG fallback); `emulator-run.sh` now extends the timeout and warns
  in that case.

  **Resolved on hardware (2026-09-22):** the Ally "freeze / can't exit" was a bug
  in the sample. The paused path did `WaitTime(0.05); continue;`, skipping
  `EndDrawing()` — and on the PlayOS raylib backend that is where
  `PollInputEvents()` (the Wayland event pump) is called. A backgrounded game
  therefore stopped servicing the compositor, which killed it ~1.5 s after the
  ARMOURY tap (`[shell] async: game crashed`), aborting the overlay/exit flow;
  no session ever logged a clean `exiting after N frames`, and two hard reboots
  followed. Fixed in samples `4e61388` by gating only the simulation on
  `paused` and always drawing + `EndDrawing()`. Written up in the sample README
  ("never skip EndDrawing()").

  **Root cause of the periodic stutter — measured, kernel-stack evidence
  (2026-09-22):** `playos-platform-api`'s evdev backend re-scans
  `/dev/input/event0-31` whenever a device class is missing. The Ally has no
  BTN_MODE "home" node, so `open_home_node()` re-runs every
  `RESCAN_INTERVAL_US` = 2 s; each scan **opens and closes up to 32 evdev
  nodes**, and evdev `close()` runs `input_close_device()` → `synchronize_rcu()`:

  ```
  __wait_rcu_gp ← synchronize_rcu_normal ← input_close_device ← evdev_release ← __fput ← close(2)
  ```

  ~30 ms per close × ~12 nodes ≈ **a ~0.4 s stall every 2 s, inside the client**.
  That one bug explains the whole symptom set: the user-visible "hitch every few
  seconds", the compositor's `100 ↔ 30` presents/s alternation (the client
  commits nothing during the stall) and the shell's ~6 % CPU.

  **Not a compositor bug:** the compositor is single-threaded and idle in
  `do_sys_poll` (CPU time does not move); a 15 s scheduling probe found no stalls
  > 10 ms (worst 3.1 ms); `dmesg` is clean; 1 Hz writes+fsync to `/data/log` are
  fast. Present path is 100 % zero-copy direct scanout, as designed.

  **FIXED — `playos-platform-api` `ff6ec10`** (pin bumped, refdistro `fd21595`).
  Discovery now opens each `/dev/input/eventN` **at most once** and keeps the
  handle in a small cache (`node_fd()`), evaluating predicates against the cached
  fd; `node_fd_release()` is the only place a device handle is closed. A scan
  costs a few ioctls and closes nothing, so the 2 s missing-class retry is
  harmless.

  **Verified on the ROG Ally, same client binary, 25 samples @ 0.4 s:**
  *before* → 5 samples in `__wait_rcu_gp`, once every **2.0 s**, ~0.4 s each
  (CPU counter frozen across each stall); *after* → **0** samples, all in
  `hrtimer_nanosleep`, CPU advancing smoothly. Gamepad + vendor-node discovery
  unchanged (still found on the first pass).

  This also explains the dead Armoury button: the shell uses the same backend,
  and the repeated close of the hid-asus vendor node (`/dev/input/event8`) stopped
  it delivering events — the shell logged its last `asus raw EV_KEY` at t=128 and
  none afterwards while still holding fd 19 open. Verified on-device by
  bind-mounting the fixed `libplayos.so.0` over the system one
  (`mount --bind /data/lib/libplayos.so.0 /usr/lib/libplayos.so.0`; **not**
  persistent — a reboot reverts it) and restarting the shell + overlay.

Caveats: the desktop raylib is X11-only unless `libdecor-0-dev` is present
(`export-sdk.sh` reports this; before this session's fix it *aborted* at the
`grep -c` instead, leaving the desktop libplayos unexported). `versions.lock`
was bumped for `init` (`4f9c599`) and `spec` (`cade621`); `platform-api`
(`f3e629c`) and `shell` (`f32727f`) still lag their repo HEADs because those S15
commits are host/SDK-side.

## Sprint 16 (`playos-net`) — next, reviewed and realigned (not started)

Wi-Fi with a D-Bus-free stack: `wpa_supplicant` + `dhcpcd` + a new trusted
`playos-net` bridge over the existing `control.sock`. **No implementation has
started.** The 2026-09-22 review corrected the sprint against the tree:
`playos_ally_defconfig` (not `playos_rog_ally_defconfig`); firmware from
`BR2_PACKAGE_LINUX_FIRMWARE_MEDIATEK_MT7921`/`_MT7922` (no overlay blobs);
wpa_supplicant via Buildroot options with `_DBUS` left unset; `dhcpcd` already
enabled; runtime IPC in `playos-init/ipc/ipc.h` + `runtime-ipc.md` (not
`protocols/`, which is the Wayland protocol); shell extends Settings
`TAB_NETWORK` + `src/screen_network.c`. The stack decision is **ADR-0012**
(`playos-spec/src/adr/ADR-0012-wifi-stack.md`). Ground truth: `board/ally/linux.config`
has `# CONFIG_WIRELESS is not set` (T1 is real work); no `playos-net` repo exists
(start it in `playos-refdistro/src/playos-net/`). T8's on-device Wi-Fi checks are
**hardware-gated** on the Ally's MT7921e; host/QEMU covers T1–T7 and the QEMU
half of T8.

## Sprint 16 — playos-net (Wi-Fi) — in progress

T1/T2/T3/T4 done; T5–T8 remain.

- **T1 kernel + firmware** — `WIRELESS/CFG80211/MAC80211/RFKILL/WLAN/MT7921E=y`
  and `LINUX_FIRMWARE_MEDIATEK_MT7922`. Verified on the installed Ally:
  `mt7921e 0000:06:00.0: ASIC revision: 79220010`, WM firmware loaded, and the
  interface comes up as **`wlp6s0`** with a `wireless/` dir, rfkill unblocked.
  (Not `wlan0`/`mlan0` — the spec was corrected.)
- **T2 packages** — wpa_supplicant with `NL80211/CTRL_IFACE/WPA3/WPA_CLIENT_SO`
  and **DBUS unset**, plus `wireless-regdb`; `dhcpcd` was already on. The shipped
  image has `wpa_supplicant`, `libwpa_client.so`, `regulatory.db(.p7s)` and
  **0 dbus entries**.
- **T3 daemon** — `src/playos-net/` (`main.c`, `wpa_bridge.c`, `profiles.c`)
  serves `/run/playos/net/bridge.sock` (`root:playos-trusted` 0660, SO_PEERCRED
  root-or-GID-1000). Runs against wpa_supplicant's control socket
  `/run/playos/net/<ifname>`; the interface is discovered from
  `/sys/class/net/*/wireless`. **Verified on hardware: `ScanNetworks` returned 10
  live networks with security + dBm.**
- **T4 IPC** — message strings in `playos-init/ipc/ipc.h` + a `Network control`
  section in `runtime-ipc.md`.

Gotchas found on the way: Buildroot's `linux-firmware` needs `-rebuild` (not
`-reinstall`) after adding a firmware option; `_WPA3=y` pulls OpenSSL; and
wpa_supplicant names its socket after the interface, so `/run/playos/net/wpa.sock`
does not exist.

Next: **T5** supervise + relay (`control.sock` → bridge, so the shell never
connects directly), **T6** the settings screen, **T7** is half-done (profiles
persist and auto-connect inside `playos-net`), **T8** E2E on hardware.

### Sprint 16 — image built with everything (2026-09-24)

`make ally-dev-usb-image` produced a 3.3 GB dev image carrying every fix from
this session, content-verified inside the shipped `rootfs.squashfs`:

- `init` — T5 supervision + control-plane relay (`network stack started`,
  `wpa_supplicant/dhcpcd/playos-net launched`)
- `usr/bin/playos-net` — the Wi-Fi bridge
- `usr/lib/libplayos.so.0` — the evdev discovery fix (permanent, no bind-mount)
- `usr/bin/playos-shell` — the ADR-0007 audio handoff
- `usr/sbin/wpa_supplicant` + MT7922 firmware + regdb
- the Invaders sample with procedural SFX

**Lesson:** a bind-mount can never activate a new `/init`. `/` is read-only
squashfs, so `mount --bind` over `/init` is lost at reboot and the old PID 1
boots. Testing a change to init *requires* a new image (or editing the EFI boot
entry's cmdline). Everything else in this list could be bind-mounted for a live
test; init cannot.

**Still outstanding:** T6's screen (`src/screen_network.c` — scan list, passphrase
entry, live status) and its `TAB_NETWORK` wiring. The T6 client helpers
(`playos_trusted_*_network*`) are implemented and build clean; only the UI
remains. T8 (connect → DHCP → WPA3 E2E, trust boundary, reboot persistence)
follows once the image is installed.

### Sprint 16 T5 — verified on hardware (2026-09-24)

First installed boot after the fix:

    [2.565] [net] waiting for a wireless interface (attempt 1)
    [4.386] [net] wpa_supplicant launched (PID 381, if=wlp6s0)
    [4.386] [net] dhcpcd launched (PID 382, if=wlp6s0)
    [4.386] [net] playos-net launched (PID 383)
    [4.386] [net] network stack started (if=wlp6s0)

- all three daemons are children of PID 1; `/run/playos/net/` holds
  `bridge.sock` + `wlp6s0` + `wpa.conf`, owned `root:playos-trusted`
- **relay works**: `ScanNetworks` sent to **`control.sock`** returned 10 live
  networks, i.e. shell → init → bridge → wpa_supplicant
- **restart works**: `kill -9 playos-net` → `playos-net exited: code=-1
  signal=9` → relaunched ~0.8 s later
- wpa_supplicant brings the interface `UP` itself (no init-side `ip link` needed)

**The bug this exposed (and why the first install failed):** init sampled
`/sys/class/net` at 2.552 s but `mt7921e` only creates the interface at 2.916 s,
so the whole network stack stayed down for the session. Fixed by retrying
discovery from the 1 Hz housekeeping tick (`playos-init` `7165356`). A one-shot
probe of anything the kernel creates asynchronously is a bug.

Also fixed while verifying: `wpa_status()` leaked uninitialised stack memory as
`"ssid"` when disconnected.

## Sprint 16 — `playos-net` — COMPLETE (2026-09-24)

T1–T8 done and verified on the ROG Ally. Wi-Fi now has a home in the product:
`wpa_supplicant` (no D-Bus) + `dhcpcd` + the trusted **`playos-net`** bridge,
supervised by init and reached only through `control.sock` (ADR-0012).

Verified on hardware, not just built:

- a scan through the control plane returns live networks; the Settings → Network
  screen lists them with signal bars and security
- **the saved profile auto-connects on boot with no ethernet dock attached** —
  reachable ~10 s after power-on (this is what makes the Wi-Fi test meaningful,
  and it also proved the wired link was never disturbed: the wired route keeps
  metric 0 while Wi-Fi sits at 3003)
- the trust boundary holds: a uid·gid 1001 probe is refused by `bridge.sock`,
  `control.sock` and `compositor.sock` (EACCES)
- init's supervision survives `kill -9` on every daemon

Two fixes found only by running it: init sampled `/sys/class/net` before
`mt7921e` created the interface (0.36 s too early), and `wpa_status()` leaked
uninitialised stack memory as the SSID when disconnected.

**T8's QEMU half is verified too:** a development-mode boot with no wireless NIC
retries at the designed cadence and gives up cleanly at 60 (`no wireless interface
after 60 tries`), never spawning a daemon. Note the trap found while testing: the
emulator image takes the install/live early-out unless a `playos-data` disk is
attached, and `output/qemu` was a stale build with no `[net]` code at all, so a
boot check against it says nothing about T5.

**Resolved 2026-09-24:** the image was flashed — the device now runs it
permanently (running `playos-shell`/`init` hashes match the built
`rootfs.squashfs` byte for byte, zero bind-mounts, and `/data` survived so the
Wi-Fi profile auto-connected again).

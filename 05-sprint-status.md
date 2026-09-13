# 05 — Sprint Status

> **Last updated: 2026-09-12** — Sprints 11.5, 11.6, 12, 13, 13.6, and 13.7 all closed (validated on-device). Sprint 14 in progress (production readiness): recovery + pause-overlay iteration landed; awaiting the on-device 19-criterion smoke/perf pass.
> Specs live in `playos-spec/src/sprints/`; this file summarizes state and evidence.

## Head SHAs (all repos clean on `main`)

| Repo | HEAD |
|---|---|
| playos-spec | `579d862` spec: ARMOURY CRATE screenshot gesture; hold-COMMAND impossible (S14) |
| playos-init | `3c7309d` init: poll-based IPC wait (removes 1s control latency) (S14) |
| playos-compositor | `f20597b` compositor: wlr-screencopy manager for screenshots (S14) |
| playos-runtime | `01c193b` runtime: playos_trusted_rollback_slot wrapper (S14) |
| playos-refdistro | `7862b3c` versions.lock: shell gesture fix + spec (S14) |
| playos-platform-api | `f3e629c` platform-api: Doxygen docs + examples + getting-started (S14 T3) |
| playos-shell | `ec0f9e5` shell: screenshots on ARMOURY CRATE tap; edge-triggered gestures (S14) |
| playos-samples | `2aaec17` fix cel shading white car |
| playos-raylib | `dbc56a8` (6.0 tag, pinned in versions.lock) |
| playos-tools | `f46f512` sdk: toolchain/pkg-config/profile scripts + docs (S15-T4 scaffolding) |
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
critical-path only because boot counting/pivot need `/EFI` (fix: pass the slot from GRUB).

Remaining in Sprint 14: the rest of P1 (initramfs/kernel ~2.5-3 s before init, shell startup ~0.9-1.4 s,
slot-from-bootloader) and P3 (direct-scanout observability).

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

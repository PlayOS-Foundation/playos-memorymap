# 05 — Sprint Status

> **Last updated: 2026-08-24** — definitive as of Sprints 11.5, 11.6, and 12 all closed (validated on-device).
> Specs live in `playos-spec/src/sprints/`; this file summarizes state and evidence.

## Head SHAs (all repos clean on `main`)

| Repo | HEAD |
|---|---|
| playos-spec | `b937a95` spec: Sprint 13 task grid progress + hardware matrix/backend portability docs |
| playos-init | `41c071b` init: Sprint 13 T5 vendor-agnostic hwmon name matchers |
| playos-compositor | `195664c` compositor: Sprint 13 T1 GPU-selection scoring extraction + unit test |
| playos-runtime | `fbc1603` runtime: control-socket policy test (S12-T6) |
| playos-refdistro | `2d0fee5` versions.lock: bump spec/compositor/platform-api/init to Sprint 13 T1+T5 commits |
| playos-platform-api | `d9d5bae` platform-api: Sprint 13 T5 vendor-agnostic hwmon readers (i915/coretemp) |
| playos-shell | `dde9f57` shell: terminate running game on B (exit fix) |
| playos-samples | `2aaec17` fix cel shading white car |
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

## Where each sprint's detail lives

`playos-spec/src/sprints/Sprint-<N>.md` — each has goal, decisions locked,
task breakdown with status grid, acceptance criteria, and handoff notes.

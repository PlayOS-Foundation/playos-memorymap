# 03 — Repositories

> Last updated: 2026-08-24. All repos clean on `main`.

| Repo | Role | Status | Key paths |
|---|---|---|---|
| **playos-spec** | Source of truth: sprints, architecture, security model | Active — S13 authored/next | `src/sprints/*.md` (22 sprints), `src/architecture.md`, `src/security-model.md`, `src/runtime-ipc.md`, `src/platform-api.md`, `src/testing.md` |
| **playos-init** | PID 1: boot, mounts, IPC server, supervision, A/B, updates, game sandbox | S12 sandbox landed | `src/main.c` (boot), `src/mount.c`, `src/supervisor.c` (spawns), `src/ipc_handler.c`, `src/boot_slot.c`, `src/update.c`, `src/thermal.c`, `src/security/*` (S12), `ipc/*` (canonical IPC) |
| **playos-compositor** | wlroots 0.20 Wayland compositor, DRM/KMS, seat input | S12 T5 landed | `src/compositor.c`, `src/drm_backend.c`, `src/renderer_gbm_egl.c`, `src/system_button.c` (reserved buttons), `src/state_machine.c`, `src/trusted_client.c` |
| **playos-shell** | Raylib launcher UI (library, settings, game detail) | Stable | `src/main.c`, `src/screen_home.c`, `src/screen_library.c`, `src/screen_game_detail.c`, `src/screen_settings.c`, `src/input.c` (evdev) |
| **playos-overlay** | Trusted in-game overlay (system button, power, brightness) | Stable | (package in refdistro; sources under `playos-refdistro/src/playos-overlay`) |
| **playos-platform-api** | Public game API (`libplayos`): input/audio/display/storage/power/system/logging/lifecycle | Stable | `include/playos/*.h`, `src/*.c`, `src/backends/backend_evdev.c` |
| **playos-runtime** | Trusted control IPC client lib + policy | S12 T6 test landed | `src/trusted_control.c`, `include/playos-runtime/trusted_control.h`, `tests/test_trusted_control.c` |
| **playos-refdistro** | Buildroot OS image build: defconfigs, packages, overlays, keys, Makefile | S12 T7/T9 landed | `br2-external/configs/`, `br2-external/board/`, `br2-external/package/`, `Makefile`, `versions.lock`, `keys/dev/`, `scripts/` |
| **playos-samples** | 11 reference games (raylib, C99) | Stable | `*/src/main.c` + `*/manifest.json` per sample |
| **playos-reference-devices** | Hardware documentation (ROG Ally) | Docs | device notes, kernel gotchas |
| **playos-tools** | Dev tooling around the platform | Early | — |
| **playos-cloud** | Cloud backend (post-MVP) | Early | — |
| **playos-marketplace** | Store/marketplace (post-MVP) | Early | — |
| **playos-foundation** | Website/org site | Docs | — |
| **playos-memorymap** | This map (docs only, not a git repo) | — | — |

## Dependency flow

```
playos-spec ──(specs)──▶ everything
playos-platform-api ──(libplayos)──▶ playos-shell, playos-samples
playos-runtime ──(trusted lib)──▶ playos-shell, playos-overlay
playos-init, playos-compositor, playos-shell, playos-overlay,
playos-platform-api, playos-raylib, playos-samples ──▶ playos-refdistro (image)
```

`playos-refdistro` consumes every OS component via local-site packages
(`br2-external/package/playos-*.mk`, `SITE_METHOD = local`) pinned by
`versions.lock`. **It also vendors a copy of the IPC sources at
`playos-refdistro/src/playos-init/ipc/` — keep in sync with
`playos-init/ipc/` (the canonical copy).**

## Which repo does what (quick answers)

- “Where do I add a syscall or change boot?” → `playos-init`
- “Where does rendering happen?” → `playos-compositor` (DRM/GBM/EGL)
- “Where is the UI?” → `playos-shell`
- “How do games talk to the OS?” → `playos-platform-api` headers only
- “Where is the IPC wire format?” → `playos-init/ipc/ipc.h`
- “Where are the kernel config / packages?” → `playos-refdistro/br2-external/board/*/linux.config`, `br2-external/package/`, `configs/*_defconfig`
- “Where are the dev keys?” → `playos-refdistro/keys/dev/`

## Per-repo agent instructions

12 repos carry an `AGENTS.md` with repo-specific rules (read it before editing
that repo). The exception is `playos-init` — its conventions are in
`06-conventions.md` and in the spec repo.

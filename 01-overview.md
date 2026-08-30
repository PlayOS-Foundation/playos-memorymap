# 01 — PlayOS Overview

> Last updated: 2026-08-24

## What PlayOS is

PlayOS is a minimal, console-style operating system for gaming handhelds.
It boots straight into a game library shell — no desktop, no package manager,
no general-purpose user accounts. The primary reference hardware is the
**ASUS ROG Ally (2023)**, with QEMU/x86_64 as the development target.

Design pillars:

1. **Single-purpose console OS** — boot to games in seconds, controller-first UI.
2. **Trusted platform / untrusted games** — games are unprivileged and sandboxed;
   the OS is a read-only, A/B-updatable image.
3. **Small, auditable core** — C99 components, static PID 1, Buildroot base.
4. **Spec-driven development** — every feature starts as a sprint doc in
   `playos-spec/src/sprints/`; code follows the spec.

## Hardware targets

| Target | Use | Notes |
|---|---|---|
| ASUS ROG Ally (2023) | primary product | AMD Ryzen Z1 Extreme, AMDGPU, Realtek ALC294 audio, USB-C |
| QEMU x86_64 + OVMF | dev / CI | `playos_qemu_x86_64_defconfig` |
| QEMU installer | installer dev | `playos_installer_qemu_defconfig` |

## Glossary

| Term | Meaning |
|---|---|
| **refdistro** | `playos-refdistro` — the Buildroot-based OS image build |
| **br2-external** | Buildroot external tree used by refdistro |
| **playos-init** | PID 1 process supervisor (initramfs) |
| **libplayos** | the only public API games may use (`playos-platform-api`) |
| **compositor** | wlroots 0.20 Wayland compositor (`playos-compositor`) |
| **shell** | the launcher UI (`playos-shell`, raylib) |
| **overlay** | trusted in-game UI overlay (system button, brightness, power) |
| **installer** | one-shot USB installer (`playos-installer` in refdistro) |
| **.playosb** | signed A/B update bundle |
| **ESP** | EFI System Partition (holds boot.json + A/B kernels) |
| **playos-data** | persistent writable data partition (`/data`) |

## Repo map at a glance

See [`03-repositories.md`](03-repositories.md) for the full table. The 13 git repos:

- **OS core:** `playos-init`, `playos-compositor`, `playos-shell`, `playos-platform-api`, `playos-runtime`
- **Distro:** `playos-refdistro` (Buildroot build + packages + defconfigs + keys)
- **Specs:** `playos-spec` (source of truth)
- **Content:** `playos-samples` (11 reference games)
- **Surrounding (mostly docs/cloud):** `playos-cloud`, `playos-marketplace`, `playos-tools`, `playos-foundation`, `playos-reference-devices`

## Where we are (one paragraph)

Sprints 0–12 are complete and validated on-device (including 11.5, 11.6, and
12). Boot, init, compositor, input, shell, storage, game launch/lifecycle,
ALSA audio, power/thermal, installer, A/B updates, squashfs pivot, developer
SSH, and the game security sandbox all work on the ROG Ally. Sprint 13
(Intel expansion) is next. See [`05-sprint-status.md`](05-sprint-status.md).

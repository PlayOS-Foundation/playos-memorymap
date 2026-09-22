# PlayOS Onboarding Guide

> **Audience:** AI agents (and humans) picking up the PlayOS codebase for the first time.
> **Last updated:** 2026-09-22 · **State:** Sprints 0–14 (incl. 11.5, 11.6, 13.6, 13.7, 14, 14.5) complete and validated on-device; **Sprint 15 (Game Developer SDK) in progress** — T1–T6 done, T7/T8 pending.

This directory is the **memory map** for the PlayOS solution under `/home/nikmes/playos`.
It is intentionally a *map*, not a duplicate of the specs — it tells you where things
live, what state they are in, how to build/test them, and what rules to follow.

---

## 1. What PlayOS is (30 seconds)

PlayOS is a console-style, Linux-based gaming OS for handhelds (primary target: ASUS ROG Ally).
It is built with **Buildroot** + a custom PID 1 (`playos-init`) + a **wlroots** Wayland
compositor + a **raylib** shell. Games are untrusted userspace processes, sandboxed with
Landlock + seccomp, talking to the platform through a single public API (`libplayos`).

- Boot: UEFI → EFI-stub kernel → initramfs → `playos-init` (PID 1) → compositor → shell
- Storage: read-only squashfs root + writable `/data` partition (`playos-data`)
- Updates: A/B slots on the ESP with signed `.playosb` bundles
- Trust boundary: `playos-init`/compositor/shell = trusted; **games = untrusted** (`playos-game`, uid 1001)

Full details: [`01-overview.md`](01-overview.md) and [`02-architecture.md`](02-architecture.md).

---

## 2. Where the truth lives

| Truth | Location |
|---|---|
| **Specs & roadmap** (source of truth for *what* to build) | `playos-spec/src/` — especially `sprints/*.md`, `architecture.md`, `security-model.md`, `runtime-ipc.md`, `platform-api.md`, `testing.md` |
| **Component pins** (exact commits used by the distro build) | `playos-refdistro/versions.lock` |
| **Per-repo rules & commands** | each repo's `AGENTS.md` (12 of 13 repos have one) |
| **This memory map** (state + how-to + gotchas) | `playos-memorymap/` (this directory) |
| **Definitive sprint status memory** | agent memory store, namespace `playos` (most recent "Sprint Status" entry) |

**Rule:** specs come first. If you change behavior, update the spec repo in the same change set.
`versions.lock` must be bumped whenever a component commit is consumed by the distro build.

---

## 3. Reading path (15 minutes)

1. [`01-overview.md`](01-overview.md) — what it is, hardware, glossary
2. [`02-architecture.md`](02-architecture.md) — boot flow, process tree, IPC, data layout
3. [`03-repositories.md`](03-repositories.md) — all 14 directories, one table
4. [`05-sprint-status.md`](05-sprint-status.md) — what is done and what is pending
5. [`04-build-and-test.md`](04-build-and-test.md) — how to build/test each repo
6. [`10-build-system.md`](10-build-system.md) — how images are generated (QEMU / Ally / installer)
7. [`06-conventions.md`](06-conventions.md) — C99, naming, commit style
8. [`07-security.md`](07-security.md) — the game sandbox (Sprint 12)
9. [`08-gotchas.md`](08-gotchas.md) — lessons learned that will bite you
10. [`09-next-steps.md`](09-next-steps.md) — how to resume work

---

## 4. Fast facts

- **14 directories, all 14 git repos** under `/home/nikmes/playos`. `playos-memorymap` is docs-only in content but is itself a git repo.
- **Language:** C99 everywhere (no C11 atomics, no VLAs). See `06-conventions.md`.
- **PID 1** is `playos-init` (static, musl), installed at `/init`. BusyBox is NOT init.
- **IPC:** Unix sockets + a framed protocol defined in `playos-init/ipc/ipc.h` (canonical; `playos-refdistro/src/playos-init` is a symlink to that repo, so there is only one copy).
- **Compositor:** wlroots 0.20 (headers/libs in `/opt/playos-deps` on this machine).
- **Games run as `playos-game` (uid/gid 1001)**, supplementary groups `audio`, `render`, `input`, sandboxed by Landlock (default-deny) + seccomp (deny-list) + `PR_SET_NO_NEW_PRIVS`.
- **All repos clean + committed** as of 2026-09-22. Head SHAs are listed in `05-sprint-status.md`.
- **Sprints 0–14 complete** (incl. 11.5, 11.6, 13.6, 13.7, 14, 14.5, validated on-device). **Sprint 15 (Game Developer SDK) is in progress:** T1–T6 done and verified; T7 (emulator profile) and T8 (reference-sample validation) pending.

---

## 5. Common tasks → where to go

| You want to… | Repo / file |
|---|---|
| Change boot/init/supervision/updates/sandbox | `playos-init` (`src/*.c`, `src/security/*.c`) |
| Change rendering/compositing/input intercept | `playos-compositor` (`src/*.c`) |
| Change the game-facing API | `playos-platform-api` (`include/playos/*.h`, `src/*.c`, `src/backends/`) |
| Change the home UI/settings/library | `playos-shell` (`src/screen_*.c`, `src/input.c`) |
| Change trusted IPC / control socket policy | `playos-runtime` (`src/trusted_control.c`) + `playos-init/ipc/` |
| Change the OS image/packages/kernel/defconfigs | `playos-refdistro` (`br2-external/`, `Makefile`, `versions.lock`) — see [`10-build-system.md`](10-build-system.md) |
| Build a QEMU / Ally / Intel / production image | `playos-refdistro` Makefile targets (`make qemu-build`, `ally-dev-usb-image`, `ally-prod-usb-image`, `intel-dev-usb-image`, …) |
| Change what should be built (specs) | `playos-spec` (`src/sprints/*.md`, `src/*.md`) |
| Look at reference games | `playos-samples` (11 sample games) |
| Hardware notes (ROG Ally) | `playos-reference-devices` |

---

## 6. Rules every agent must follow

1. **Spec-first.** Read the relevant sprint doc; update it in the same change set as code.
2. **Never skip tests.** Every C change gets a host test where feasible (`ctest`).
3. **C99 + conventions** (`06-conventions.md`): `playos_` public, `playos__` internal, `playos_ipc_` IPC, one state struct per process, no dynamic allocation in hot paths.
4. **Keys/credentials:** never commit production keys. Dev keys live in `playos-refdistro/keys/dev/` only.
5. **Pins:** if your commit will be consumed by the distro build, bump `versions.lock` (edit it directly — the `scripts/update-versions.sh` helper referenced in its header does not exist yet) and commit it in `playos-refdistro`.
6. **Commits per repo** with a `subsystem: summary` message; push to `main` (all PlayOS repos use `main`).
7. **Hardware-blocked ≠ done.** QEMU/host validation is necessary but not sufficient; mark the on-device acceptance separately.

---

## 7. If you are resuming autonomous work

Start at [`09-next-steps.md`](09-next-steps.md). The next sprint is **Sprint 15 (Game Developer SDK)** and it is
already underway — read `playos-spec/src/sprints/Sprint-15.md` (spec head `0baf5d1`; T1–T6 done, T7 emulator
profile and T8 reference-sample validation remain) and follow the spec-first workflow.
`PLAYOS_SPEC_COMMIT` in `playos-refdistro/versions.lock` lags HEAD (`4e8dbf5`) because the S15 commits are
host/SDK-side; re-check the pin before a release.

---

*Map maintained as plain Markdown. If a file contradicts `playos-spec` or `versions.lock`, those win — and please fix the map.*

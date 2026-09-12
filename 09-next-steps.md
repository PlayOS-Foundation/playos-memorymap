# 09 — Next Steps & How to Resume

> Last updated: 2026-09-12

## Where we are

Sprints 0–13.7 are complete and validated on-device (including 11.5, 11.6, 12,
13, 13.6, 13.7). Sprint 14 (Production Readiness) is in progress. Read
[`05-sprint-status.md`](05-sprint-status.md) for the current head SHAs and
evidence, and `playos-spec/src/sprints/Sprint-14.md` for the task grid.

## Next up: finish Sprint 14, then Sprint 15

Sprint 14 remaining work — all hardware-gated (see Sprint-14.md):

1. **19-criterion MVP smoke test** on the ROG Ally (`scripts/mvp-smoke.sh`) —
   T5.
2. **Performance baseline** run (`scripts/perf-baseline.sh`) — T7; targets are
   cold boot to shell < 5s, shell→game first frame < 3s, SYSTEM→overlay < 100ms,
   game exit→shell < 500ms.
3. **SimpleDRM / low-graphics recovery** validation — recovery must render
   without AMDGPU (T6 acceptance gap).
4. **T10 installer-as-an-app front-end** polish; **T9** final signed v0.3.0 run
   + SDK-compile verification.

Validation loop for the current recovery/overlay work: build
`make ally-dev-usb-image`, flash, and check the recovery entry points plus
COMMAND/SYSTEM pause-overlay behaviour. Fixes to verify on-device:
`RollbackSlot` rollback and the non-blocking recovery watch (init `0094e83`,
runtime `01c193b`, shell `1bc7403`), plus the overlay fixes — hidden overlay
ignores the gamepad, d-pad is decoded from `ABS_HAT` on the Ally, and the pause
menu is a focus list (Resume / Quit / Profile) where B resumes and Quit needs a
held A (refdistro `8424797`, compositor `6fd8f63`, spec `f888d29`).

Sprint 15 (Game Developer SDK) has already been scaffolded in `playos-tools`
(`f46f512`) and `playos-refdistro` (`scripts/export-sdk.sh`, `2e5fadc`); it
becomes the active sprint once Sprint 14's hardware gate closes.

## Open follow-ups

1. **SimpleDRM / low-graphics recovery (F3).** Recovery still renders through
   the compositor, so the compositor-failure entry point cannot show the menu
   when graphics are what broke. Needs a software/SimpleDRM render path
   (S14-T6 acceptance gap).
2. **11.5 installer `wipefs` follow-up (non-blocking):** doesn't reliably clear
   the inactive slot on reinstall; fresh installs still pivot correctly.

Resolved on 2026-09-12: F1 (Rollback corrupted `boot.json`), F2 (4 s recovery
watch on every boot), F4 (hidden overlay consumed gamepad input / Ally d-pad
not decoded).

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

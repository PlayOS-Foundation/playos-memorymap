# 09 — Next Steps & How to Resume

> Last updated: 2026-08-30

## Where we are

Sprints 0–13 are complete and validated on-device (including 11.5, 11.6, 12,
and 13). Read [`05-sprint-status.md`](05-sprint-status.md) for the current head
SHAs and evidence. Sprint 14 (Production Readiness) is next — see
`playos-spec/src/sprints/Sprint-14.md`.

## Next up: Sprint 14 (Production Readiness)

1. Read `playos-spec/src/sprints/Sprint-14.md` first — it freezes the public
   `libplayos` API, automates the signed release pipeline, and smoke-tests the
   preview release on ROG Ally.
2. Follow the spec-first workflow in [`06-conventions.md`](06-conventions.md):
   implement, update the spec in the same change set, then validate in QEMU and
   (where possible) on the ROG Ally target.

## Housekeeping

1. ~~**Bump the stale `versions.lock` pin.**~~ **Done 2026-08-24:** `PLAYOS_SPEC_COMMIT`
   is pinned to `29dfd08` (Sprint 12 on-device 11/11 self-test closed) and
   committed in `playos-refdistro`.
2. **Open 11.5 follow-up (non-blocking):** the installer's `wipefs` doesn't
   reliably clear the inactive slot on reinstall. Tracked; fresh installs still
   pivot correctly.

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

# 06 — Coding Conventions

> Last updated: 2026-08-22. These rules apply to **all C repos**.

## Language

- **C99 only.** No C11 atomics, no VLAs, no `//`-only files with mixed styles is fine but be consistent per file.
- `-Wall -Wextra` (plus `-Wpedantic` in playos-init). Zero new warnings.
- musl-compatible; `playos-init` is **static** (no shared libs, no NSS, no `getgrnam`).

## Naming

| Scope | Convention | Example |
|---|---|---|
| Public API | `playos_` prefix | `playos_input_get_controller_state()` |
| Internal symbols | `playos__` double underscore | `playos__backend_open()` |
| IPC symbols | `playos_ipc_` prefix | `playos_ipc_send()` |
| Shell screens | `screen_NAME_enter/update/draw()` | `screen_home_draw()` |
| Constants/macros | `PLAYOS_` UPPER_SNAKE | `PLAYOS_API_VERSION` |

## State & memory

- **One global mutable state struct per process** (`struct playos_init_state g_state`,
  `struct playos_compositor c`, etc.). No other global mutable variables.
- **No dynamic allocation in hot paths** (input polling, audio write, render loop).
- Manifest/update parsing allocates sparingly and always frees.
- No `exit()`/`abort()` except on unrecoverable init failure. Games/shell get
  killed by the supervisor, not by self-destructing.

## Process rules

- `playos-init` is PID 1: it must never exit on child failure; supervise + restart.
- Drop privileges **before exec, not after** (Sprint 12 sandbox ordering).
- Log via `playos_ipc_log()`/`playos_log_write()` — not `fprintf` to stdout in
  components that have the logging API.

## UI rules

- Shell UI dimensions are **fractions of screen size**, never hardcoded pixels.
- Colors/fonts follow the existing `render_util.c` helpers.

## Spec-first workflow

1. Read the relevant sprint in `playos-spec/src/sprints/`.
2. Change the spec first (task grid, acceptance criteria).
3. Implement code.
4. Add host tests where feasible.
5. Build + `ctest`.
6. Commit per repo with `subsystem: summary`; push to `main`.
7. If consumed by the distro, bump `versions.lock` (edit directly — no helper script yet).

## Commit style

```
<repo or subsystem>: <imperative summary>

- bullet points of what/why (optional for small changes)
```

Examples: `init: Sprint 12 game sandbox (T1-T4, T8)`, `spec: Sprint 12 status + security-model updates`,
`compositor: full reserved-button intercept (S12-T5)`.

## Security rules

- Never commit production secrets. Dev keys only under `playos-refdistro/keys/dev/`.
- Games are untrusted: anything a game can touch must be validated (manifest fields,
  game_id strings, save paths).
- New sandbox paths go through the data-driven path construction in
  `playos-init/src/security/landlock.c` (Sprint 21 will extend it for profiles).

# 02 — Architecture

> Last updated: 2026-08-24

## Boot flow (ROG Ally / QEMU)

```
UEFI firmware
  └─ EFI/BOOT/BOOTX64.EFI      (Linux EFI stub; kernel + embedded initramfs)
       └─ kernel 6.12.103
            └─ /init = playos-init   (static musl PID 1)
                 │ 1. mount proc/sys/dev/tmpfs
                 │ 2. find & mount ESP (/EFI), read boot.json → active slot
                 │ 3. find & mount playos-data → /data  (prefers removable;
                 │    falls back to internal NVMe only on installed systems)
                 │ 4. create /data dirs, seed shipped games on first boot
                 │ 5. IPC: /run/playos/control.sock + compositor.sock
                 │ 6. spawn compositor → wait for readiness
                 │ 7. spawn shell + overlay  (or installer in install mode)
                 │ 8. optionally spawn playos-ssh-bringup (dev image only)
                 │ 9. supervise everything; reap zombies; restart policy
                 ▼
            playos-compositor (wlroots DRM backend)
                 └─ playos-shell (raylib Wayland client)
                 └─ playos-overlay (trusted in-game UI)
```

Installer mode is selected by kernel cmdline `playos.mode=install` (installer image).

## Process model

| Process | Binary | User | Privileges | Notes |
|---|---|---|---|---|
| PID 1 | `/init` → `playos-init` | root | full | supervises everything |
| Compositor | `/usr/bin/playos-compositor` | root | full (for now) | DRM master, seat input |
| Shell | `/usr/bin/playos-shell` | root | full (for now) | raylib UI; reads `/dev/input/event*` directly |
| Overlay | `/usr/bin/playos-overlay` | root | full (for now) | trusted in-game UI |
| **Game** | `/data/games/<id>/<exe>` | **playos-game** (1001:1001) | **none** | Landlock + seccomp + `NO_NEW_PRIVS`; groups `audio`, `render`, `input` |
| Installer | `/usr/bin/playos-installer` | root | full | installer image only |
| SSH bring-up | `/usr/bin/playos-ssh-bringup` | root | full | dev image only (Dropbear) |

## IPC map

| Socket | Path | Owner/mode | Who connects |
|---|---|---|---|
| Control | `/run/playos/control.sock` | `root:playos-trusted` `0660` | shell, overlay (gid 1000 or uid 0) |
| Compositor control | `/run/playos/compositor.sock` | same policy | runtime/init side |
| Wayland | `/run/playos/playos-0` (`WAYLAND_DISPLAY`) | compositor | shell, overlay, games |
| Lifecycle pipe | `PLAYOS_LIFECYCLE_FD` env | init→game | game reads single-byte events |

Wire format + message types: `playos-init/ipc/ipc.h` (canonical) and
`playos-spec/src/runtime-ipc.md`. Game launch is initiated by the shell via
`LaunchGame`/`ApplyUpdate`-style JSON messages over the control socket;
`playos-init` parses the manifest, verifies the signature (warn-only), forks
the game with the full sandbox, and reports lifecycle events back.

## Storage layout

```
ESP (vfat, mounted /EFI)
├── boot.json                    # active slot + strike counters (A/B)
└── playos-a/ playos-b/          # kernel+initramfs per slot

/data (ext4, label playos-data, mounted /data)
├── games/<game-id>/             # seeded from /usr/share/playos/games on first boot
├── saves/<game-id>/             # per-game saves (0700 playos-game)
├── cache/<game-id>/             # per-game cache (0700 playos-game)
├── config/                      # system config (thermal.json etc.)
├── log/                         # persistent logs (init, compositor, games, ssh)
├── ssh/                         # Dropbear host keys + authorized_keys (dev)
└── system/, profiles/, resources/, downloads/, updates/, screenshots/

/ (squashfs, read-only)          # rootfs; persistent state NEVER lives here
/run (tmpfs)                     # sockets, boot-stage marker, runtime state
```

Rootfs overlay sources: `playos-refdistro/br2-external/board/common/rootfs-overlay`
(shared) and `board/dev/rootfs-overlay` (dev-only: SSH bring-up). The production
image uses only the common overlay.

## The game sandbox (Sprint 12)

Applied in the game child process **before exec**, in this order:

```
1. prctl(PR_SET_NO_NEW_PRIVS, 1)
2. Landlock ruleset (default-deny allowlist)  — while still root
3. capability drop + setgroups(3) + setgid(1001) + setuid(1001)  # groups: render, audio, input
4. seccomp-BPF deny-list
5. chdir(game_dir) + exec
```

Full details in [`07-security.md`](07-security.md) and
`playos-spec/src/security-model.md`.

## Key specs to read

- `playos-spec/src/architecture.md` — canonical architecture
- `playos-spec/src/runtime-ipc.md` — IPC protocol
- `playos-spec/src/platform-api.md` — public game API
- `playos-spec/src/partition-layout.md` — disk layout
- `playos-spec/src/security-model.md` — trust model (updated for Sprint 12)
- `playos-spec/src/kernel-config.md` — kernel config decisions

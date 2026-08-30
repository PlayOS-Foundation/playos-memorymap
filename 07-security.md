# 07 — Security Model (Sprint 12)

> Last updated: 2026-08-24. Canonical doc: `playos-spec/src/security-model.md`.
> Implementation: `playos-init/src/security/`.

## Trust zones

| Zone | Members | Trust |
|---|---|---|
| Trusted | `playos-init`, compositor, shell, overlay, installer | root, full capabilities (component-level drop deferred) |
| Untrusted | games | `playos-game` (uid 1001, gid 1001), no capabilities, sandboxed |

Games may only use the public `libplayos` API (`playos-platform-api`) and the
Wayland seat. Direct control sockets, other games' data, and the primary DRM
node are denied by OS enforcement, not convention.

## Game sandbox — applied in the child before exec (order matters)

```
1. prctl(PR_SET_NO_NEW_PRIVS, 1)      # no setuid/file-cap gain ever
2. Landlock default-deny allowlist      # while still root (needs no_new_privs)
3. capability drop + setgroups(3)       # drop caps; set groups {render, audio, input}
   + setgid(1001) + setuid(1001)
4. seccomp-BPF deny-list                # after setuid — filter denies setuid/setgid
5. chdir(game_dir) + exec
```

If Landlock is unsupported (kernel < 5.13) or seccomp install fails, the
launch continues with a loud warning — but the **credential drop always
hard-fails the launch** (`_exit(126)`) if it cannot be performed.

## Landlock allowlist (default-deny)

| Path | Access |
|---|---|
| `/data/games/<id>/` | read + execute |
| `/data/saves/<id>/`, `/data/cache/<id>/` | read + write + create + remove + truncate (dirs pre-created `0700 playos-game`) |
| `/tmp` | read + write + create + remove |
| `/run/playos/` | read + execute (Wayland socket path) |
| `/lib`, `/usr/lib` | read + execute (musl loader + shared libs — games are dynamically linked) |
| `/dev/snd` | read + write (ALSA) |
| `/dev/input` | read (event devices — optional) |
| `/dev/dri` | read + write (render node only — optional) |
| `/dev` | listable (device-dir enumeration for opendir — optional) |
| `/dev/shm` | read + write + create + remove (wl_shm fallback) |
| `/sys` | read (libdrm/Mesa resolve render nodes — optional) |
| `/etc/asound.conf` | read (optional single-file rule) |

Everything else is denied — notably `/data/config`, other games' dirs,
`/proc`, `/dev/dri/card*` (the primary node, via the `drm` group), and the
control sockets. `/dev`, `/sys`, `/dev/input`, and the `/dev/dri` directory are
granted at the directory level only (listable/readable), not as full read-write.

## seccomp deny-list (default-allow, curated denies)

Denies the privileged/credential syscalls: `mount`, `umount2`, `pivot_root`,
`chroot`, `acct`, `swapon`, `swapoff`, `quotactl`, module load, `setuid`/
`setgid` family, `capset`, `ptrace`, `process_vm_*`, `reboot`, `kexec_*`,
clock/time/domain setters, `iopl`/`ioperm`, `personality`, `vhangup`, `mknod`,
`unshare`, `setns`, `seccomp`, `bpf`, `perf_event_open`, `userfaultfd`,
keyring, `open_by_handle_at`, and **`prctl(PR_SET_SECCOMP)`** (arg-checked).

A full syscall *allowlist* is deliberately deferred: games are dynamically
linked (musl + raylib + libplayos) and a hand-maintained allowlist is too
regression-prone for the MVP. Landlock default-deny is the path boundary;
the seccomp deny-list blocks the privileged syscalls Landlock can't reach.

## Input boundary

- Compositor consumes reserved buttons at the seat layer:
  SYSTEM = `BTN_MODE`/`KEY_PROG1`/`BTN_TRIGGER_HAPPY1`;
  QUICK_MENU = `KEY_PROG2`/`BTN_TRIGGER_HAPPY2`.
- Games **are** in the `input` group and `/dev/input` is granted read-only, so
  the boundary is the combination of: (a) `libplayos` bitmask stripping of
  reserved buttons, and (b) the compositor's seat-level intercept — not raw
  path denial.
- The ASUS EC device `0b05:1abe` is moved to `root:root 0600` by the
  `99-playos-input.rules` udev rule so games can't read it even with `input`
  group membership.

## IPC boundary

- `/run/playos/control.sock` and `compositor.sock`: `root:playos-trusted` `0660`.
- Peer check (`SO_PEERCRED`): accept gid 1000 or uid 0; everyone else rejected.
- Games run as uid/gid 1001 with no `playos-trusted` membership → cannot connect.

## Identity & groups (refdistro)

- `playos-game` user/group: uid/gid **1001** (`board/ally/users-table.txt`).
- `/etc/group` overlay adds `playos-game:x:1001:` and the supplementary groups:
  `audio:x:29:playos-game`, `render:x:108:playos-game`,
  `input:x:102:playos-game`.
  ⚠️ The overlay **replaces** the generated `/etc/group` — edit the overlay file,
  not the Buildroot users table, for group memberships.
- `playos-trusted` group: gid **1000** (shell/overlay accepted via uid 0 for now).

## Manifest signing (warn-only)

- `manifest.json.sig` = raw 64-byte Ed25519 detached signature over `manifest.json`.
- Public key embedded in `playos-init/src/security/game_key.h`
  (dev key: `keys/dev/manifest-key.pub` in refdistro).
- Verify is **warn-only**: invalid/missing signatures log a warning; launch proceeds.
- Sign with `playos-refdistro/scripts/sign-manifest.sh`.

## Secure Boot (foundations)

- Target chain: UEFI Secure Boot → `BOOTX64.EFI` → kernel → dm-verity/IMA rootfs → signed `.playosb`.
- Dev keys: `playos-refdistro/keys/dev/{efi-signing-key.pem,efi-signing-cert.pem}`,
  sign via `scripts/sign-efi.sh` (`sbsign`). Production = HSM, post-MVP.

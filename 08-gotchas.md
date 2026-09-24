# 08 — Gotchas & Lessons Learned

> Last updated: 2026-09-12. These are the things that have already bitten
> someone on this project. Read before debugging.

## Wayland

- **A NULL listener slot aborts the process.** libwayland-client prints
  `listener function for opcode N of <interface> is NULL` and then calls
  `abort()`. It is not a warning. This killed the shell on hardware: we bound
  `zwlr_screencopy_manager_v1` at **version 3** but only filled the version-1
  events in the generated frame listener, so the compositor's `linux_dmabuf`
  offer (opcode 5) hit a NULL slot → SIGABRT → shell dead → empty compositor
  scene ("blue screen"). Two rules: **fill every member of a generated listener
  struct**, and **bind the protocol version you actually implement** (we now
  bind screencopy at v1, where wl_shm is guaranteed and the client copies
  directly from the `buffer` event; v3 requires waiting for `buffer_done`).
- **Honour the `wl_shm` format — it is not always `XRGB8888`.** The screencopy
  `buffer` event names the format the compositor will fill, and the byte order
  follows it: little-endian `XRGB8888`/`ARGB8888` memory is `B,G,R,X`, but
  `XBGR8888`/`ABGR8888` memory is `R,G,B,X`. Assuming `B,G,R,X` unconditionally
  swapped red and blue on hardware (the UI's navy `(20,41,76)` background was
  written as `(76,40,20)`, orange highlights came out blue). Verify colour
  paths by sampling a PNG pixel and comparing with the colour the UI source
  draws, not by eye.
- **Roles must be released on disconnect.** The compositor kept `shell_client`
  set after the client died, so the supervisor's restart was rejected with
  `shell role already taken` and ran untrusted for the whole session. Fixed with
  `wl_client_add_destroy_listener` (`3862e4d`). Any time a trusted role is
  claimed, add a destroy handler for it.
- **Append-only logs hide boot boundaries.** `/data/log/shell-stderr.log`
  survives restarts, so a crash is found via `grep -n "entering main loop"` to
  locate boots and `init.log`'s `shell PID N exited: code=-1 signal=6`
  (signal 6 = SIGABRT) to confirm a crash.
- **"Suspended" stops drawing, not logic.** The shell's screen `_update()`
  handlers used to run while a game was foreground, so they reacted to buttons
  the game was using — the game-detail screen's B ("back") calls
  `playos_trusted_terminate_game()`, and pressing B during gameplay killed the
  game (`shell 639b73d`). Gate every input-consuming loop on `!is_suspended`;
  the only things that may act mid-game are the reserved-button gestures and
  the volume keys. When a game dies, read *how*: `signal=15` means something
  else terminated it (`TerminateGame from fd=N` in `init.log` — fd 10 is the
  shell's control connection), `code=0` means the game exited on its own.
- **The live USB and an installed disk share every partition name.** Both use
  `ESP`, `playos-a`, `playos-b`, `playos-data` as GPT names, so *anything* that
  resolves a partition by name is ambiguous once an install exists — and the
  first match in `/proc/partitions` is the NVMe. On the Ally a USB boot picked
  the NVMe's ESP (no `EFI/playos/live-usb` marker there) and pivoted into the
  *installed* slot, so "boot from USB" silently booted the installed system and
  the installer option never appeared. The unambiguous signal is the firmware's
  own record: `BootCurrent` → `Boot####` → device path (USB node = type 3
  subtype 5/0x10), which is what `init` now consults
  (`src/boot_media.c`, host-tested). Corollary: a live boot must not touch the
  installed slot's `boot.json` counters, or repeated live boots can trip its
  3-strike rollback. Related trap: the install-payload check used to
  `mkdir("/mnt/...")`, which fails with EROFS when the root is the installed
  squashfs — mount check points under `/run`.

## Rendering and recovery

- **A boot decision that reads `/proc/cmdline` must run after
  `playos_mount_virtual()`.** `playos.recovery` was checked before `/proc`
  existed, so it silently returned 0 on *every* boot; the button-hold entry uses
  evdev and therefore worked, which hid it. If a boot flag "does nothing", check
  when it is read. (init now logs the whole cmdline at boot.)
- **A headless backend still needs a renderer and an allocator.**
  `wlr_output_init_render()` asserts `allocator != NULL && renderer != NULL` and
  aborts the process (SIGABRT, "Assertion failed ... types/output/render.c"), so
  a "fallback" that creates only the backend crash-loops the compositor.
- **Create the wlroots backend exactly once.** `wlr_renderer_init_wl_display()`
  publishes `wl_shm`/linux-dmabuf globals; retrying a *failed* backend start with
  a fresh renderer duplicates them and ends in an abort. Probe the device first
  (`playos_output_select_from_fd()` on each `/dev/dri/card*`), then create once.
- **`CONFIG_DRM_SIMPLEDRM` alone creates no card** — `CONFIG_SYSFB_SIMPLEFB`
  (plus `CONFIG_FB`) registers the firmware-framebuffer platform device it binds
  to. Both are now on for ally and qemu; SimplEDRM is the display of last resort
  when the GPU driver never probes.
- **wlroots' pixman renderer is available** (compiled into `libwlroots-0.20`):
  `WLR_RENDERER=pixman` plus the DRM backend drives it through dumb buffers, so
  software rendering still reaches a real display. Recovery uses it via
  `PLAYOS_RENDERER=pixman`.
- **Testing "no GPU driver" without a device:** boot QEMU with `-vga cirrus`
  (this kernel has no cirrus DRM driver) so the guest's only DRM device is
  SimplEDRM, then read the emulated screen with the QEMU monitor's `screendump`
  (`scripts/qemu-recovery-check.sh`). Guest logs come off `output/qemu/images/data.img`
  with `mount -o ro,noload` (the killed QEMU leaves the ext4 dirty).

## Rendering and recovery

- **A GL client cannot run on a software-rendered compositor.** Forcing
  `WLR_RENDERER=pixman` for recovery starts the compositor but Raylib's EGL then
  fails (`failed to get driver name for fd -1` → `eglInitialize 0x3001`): with
  the pixman renderer there is no dmabuf/GL, so the shell exits, crash-loops five
  times and init leaves an empty compositor on screen (a blue screen). Recovery
  must keep the accelerated renderer; a recovery UI that needs no GL at all
  (a `wl_shm` client) is the right answer for the no-GPU case. This image also
  ships no `/usr/lib/dri` software rasteriser, so `LIBGL_ALWAYS_SOFTWARE=1`
  cannot rescue it either.

## Rendering and recovery

- **A GL client cannot run on a software-rendered compositor, so recovery needs a
  GL-free UI.** Forcing `WLR_RENDERER=pixman` starts the compositor but Raylib's
  EGL then fails on the Ally (`failed to get driver name for fd -1` →
  `eglInitialize 0x3001`); the shell exits, crash-loops and init leaves an empty
  compositor (a blue screen). `playos-recovery` (new `playos-recovery` package)
  is the answer: a `wl_shm` client that rasterises its own text, reads evdev
  directly, and performs the menu actions over the trusted IPC. init starts it
  when the shell hits its restart limit *in recovery*; the GL shell stays the
  preferred UI whenever it works.
- **Hooks for testing the broken-graphics path without a broken GPU:**
  `playos.renderer=pixman` (compositor software rendering) and `playos.noshell=1`
  (the shell fails on purpose). Both are kernel-cmdline options honoured by init;
  used by `scripts/qemu-recovery-check.sh` (F3_APPEND).

## Debugging a crash

- **A crash can be invisible because the child's stderr is block-buffered.**
  `child_log_redirect()` points stderr at a file, so stdio buffers it and a
  SIGSEGV takes the last lines with it — the compositor's log looked like it
  "stopped after the DRM modeset". Now line-buffered in init; keep it that way.
- **Fault-logger technique (no reflash needed):** the compositor defaults to a
  headless backend, so you can reproduce and instrument on the running device:
  cross-compile a tiny `LD_PRELOAD` .so that caches the main-object load base at
  constructor time, installs a `SA_SIGINFO` handler and writes
  `sig/fault/rip/rsp/rbp/caller` with raw `write(2)` (no `dladdr` *inside* the
  handler — loader calls there can fault or deadlock, which silently loses the
  report). Run `PLAYOS_BACKEND=headless XDG_RUNTIME_DIR=/tmp/diag
  LD_PRELOAD=... playos-compositor`, drive it with the real shell/overlay, then
  map `caller - lib_base + file_off` from `/proc/<pid>/maps` through
  `addr2line -f -e <lib>` on the build host.
- **`rip=0x0, fault=0x0` means a call through a NULL function pointer** — in
  Wayland code that is almost always a `wl_listener` whose `notify` was never
  set, or an init function that was never called. Check both.

## Boot medium, A/B slots and the live image

- **The live USB image is live-only, by an explicit switch.** A stick's ESP
  carries `EFI/playos/live-usb` (the medium marker) and `EFI/playos/live-boot`
  (the switch); init refuses to pivot when it sees the switch on the same ESP
  that carries the marker, and stays in the initramfs — which *is* the live
  system. Both files are stamped by `scripts/gen-*-usb-image.sh`, and the
  installer copies only `BOOTX64.EFI` to the target ESP, so an installed system
  never inherits the switch.
- **The live medium is identified under the pivot by its own ESP**: removable
  disks are scanned for an ESP carrying `EFI/playos/live-usb`
  (`playos_removable_esp_has_live_marker()`); when one is present init stays in
  the initramfs. That check needs a **bounded wait** — the stick enumerates at
  ~4.1 s while the pivot decision runs at ~2.1 s (measured), and the boot-time
  shortening of the ESP stage is what removed the accidental 5.5 s cover that had
  made live boots work. Gated on `playos_booted_from_usb() != 0`, so an installed
  boot (a registered, non-USB boot entry) does not wait.
- **The `removable` sysfs flag IS usable**: a SanDisk stick reports `removable=1`
  both directly and behind a dock/hub. An earlier note here claimed the dock
  reported 0 — that was wrong, inferred from a `by-label` symlink that alternates
  between the stick and an installed disk which share filesystem labels.
- **`BootCurrent` is not usable on this firmware**: the Ally reports
  `BootCurrent = 0x0006` while only `Boot0000-0002` exist (the stick is
  `Boot0002`), because it boots removable media through its fallback path and
  never persists the transient entry. `playos_booted_from_usb()` therefore
  cannot be the primary signal for the boot medium.
- **A live boot still reads the installed disk's boot.json** while `/EFI` is the
  name-found (internal) ESP, so its boot accounting can advance the installed
  slot's `boot_count`. Gate that accounting on the same live-medium check.
- **efivarfs is not auto-mounted** (no systemd), and `playos_booted_from_usb()`
  needs it; init mounts it best-effort in `playos_mount_virtual()`.

## Power and idle behaviour

- **Analog axes are never quiet.** The Ally's right stick rests with a +/-128
  oscillation on `ABS_RY` (measured: 511 <-> 767, ~65 events/s with nobody
  touching it). Any idle/activity heuristic built on *raw evdev traffic* will
  therefore think the user is constantly active. Count **discrete** input
  (EV_KEY, d-pad hat) as activity; if analog must count, use a delta threshold
  well above the drift (the observed step is 256 counts).
- **Skipping a frame also skips raylib's frame pacing.** raylib waits for the
  target FPS inside `EndDrawing()`, so an idle loop that skips drawing spins and
  burns a core unless it sleeps itself (the shell sleeps 4 ms, which keeps input
  latency in single-digit ms). Related: a screen whose *purpose* is watching
  analog values (the Live Input Test) must ask for full frame rate explicitly.

## Build system

- **Buildroot does not reliably rebuild a `local`-site package when its source
  changes.** Editing `src/playos-recovery/main.c` and running `make ally-dev-usb-image`
  produced an image with the *previous* binary: the bundle shipped and the device
  showed the old behaviour until `playos-recovery-rebuild` was run explicitly.
  After touching a local package's source, always run
  `<pkg>-rebuild` for **each** O= output you care about, then verify the binary
  inside the produced image (`mount -o loop,ro images/rootfs.squashfs` + hash
  against `target/`) *before* bundling or flashing.
- **Adding a `linux-firmware` option needs `-rebuild`, not `-reinstall`.**
  `linux-firmware` selects files at **build** time and bakes them into
  `br-firmware.tar`; the install step only extracts that tarball. So enabling
  e.g. `BR2_PACKAGE_LINUX_FIRMWARE_MEDIATEK_MT7922` and re-running the image
  build silently produced an image **without** the Wi-Fi firmware, and
  `linux-firmware-reinstall` re-extracted the stale tarball and changed nothing.
  `make linux-firmware-rebuild` (or `-dirclean`) regenerates the tarball; verify
  with `find output/<target>/target/lib/firmware -iname '*MT7922*'` before
  flashing. Same class of trap applies to any package whose install list is
  computed from its Kconfig options.
- **A new package option may pull a large dependency you did not plan for.**
  `BR2_PACKAGE_WPA_SUPPLICANT_WPA3=y` selects **OpenSSL**, which added ~10 min
  and ~3 MB to the image. Expected (SAE needs a real crypto backend), but it is
  worth knowing before wondering why the build suddenly compiles OpenSSL.

## Partition and filesystem work

- **Never parse a partition out of a device name — ask sysfs.**
  `nvme0n1` *is* a disk that ends in a digit, so "strip the trailing number"
  turns it into `nvme0` while `nvme0n1p1` becomes `nvme0n1`: the two never match.
  That silently stopped the installer handoff from releasing `/EFI` (it decided
  the ESP was "not on the install target"). Use
  `/sys/class/block/<dev>/partition` (exists only for partitions) plus the
  resolved parent directory — verified on the Ally: `nvme0n1p1 -> nvme0n1`,
  `sda1 -> sda`, `nvme0n1 -> nvme0n1`.
- **mkfs refuses a mounted device** with exit 1 and
  `<dev> contains a mounted filesystem`; the kernel also cannot re-read a
  partition table that is in use. So release every mount on the target before
  repartitioning — and do not rely on someone else having done it:
  the installer calls `playos_format_release_target()` before step 0.
- **Keep the tools' stderr.** `run_cmd()` used to send it to `/dev/null`, so a
  failure reduced to "mkfs.fat failed (exit 1)" and cost hours of guessing. It
  now captures (and drains) the child's output into the error string, which the
  installer logs and shows on screen.

## Installer handoff

- **Ack a synchronous IPC request before you start tearing the caller down.**
  `playos_trusted_start_installer_target()` is a `send_and_recv()`, so the shell
  blocks in `recv()` for `StartInstallerAck`. init used to send that ack *after*
  the handoff, and the handoff waits for the shell to exit — the pair stalled for
  the whole 2 s timeout, and by the time the old shell's Wayland client finally
  disconnected the installer had registered, been rejected with
  `shell role already taken` and aborted (`eglInitialize failed`). The ack now
  means "accepted, handing over now" and goes out first. Same rule for any future
  trusted call whose handler stops the caller.
- **The compositor only frees a trusted role when the client's socket closes**, so
  a handoff that replaces a client must ensure the old process is *gone* before
  spawning its replacement: SIGTERM then SIGKILL after 800 ms
  (`stop_client_hard()`), plus a short settle. The overlay is the first client
  the handoff kills and the shell handles SIGTERM (so it can outlive the wait),
  which is how this bit. The compositor now logs the holder pid on a rejected
  claim and every role release — read those lines first when a handoff misbehaves.


- **Do not tear the session down to install.** The S13.7 handoff stopped the
  compositor only so `/data` could be unmounted; that cost a DRM modeset blink
  (a black flash) and silently discarded the installer's log. Now only the UI
  clients make way, the compositor keeps running (the installer claims the shell
  role, freed when the shell dies) and `/data` stays mounted; `/EFI` is released
  only when it is on the install target. A failed install hands the session back
  to the shell instead of rebooting.
- The installer must **look like the shell** (same Silkscreen font, navy
  palette, 15% column) — otherwise the handoff reads as a different app.

## Shell rendering

- **Every screen must clear the frame first** (`render_begin_frame()`), because
  the renderer keeps the previous frame's pixels. A screen that forgets is drawn
  *on top of* the previous screen — the installer screen appeared over Settings
  with both fully legible on hardware. The installer and recovery screens were
  the two offenders.
- Screen layout conventions live in `playos-spec/src/playos-shell-spec.md`
  ("Layout Conventions"): content column `0.15 * width`, row rhythm
  `8 * label_scale`, right-aligned affordance on selectable rows, hint line at
  `height - scale * 45` (above the status bar), and the three-zone status bar
  (battery / temperatures / profile + thermal). Reach for those instead of
  inventing per-screen spacing; the settings screen had drifted (16*scale info
  rows, a tab strip inset from the column, a half-empty selection bar).

## Input / reserved buttons

- **The Ally's reserved buttons are momentary pulses — no hold gestures.** A raw
  evdev trace (2026-09-12) shows the Command Center button emitting `KEY_F16`
  press **and** release within the same shell poll (~1 ms apart), and Armoury
  Crate emitting `KEY_PROG1` the same way. Physically holding either button
  changes nothing, so `shell_input_button_held()` can never observe them — a
  "hold COMMAND" gesture is impossible on this hardware. Use the edge query
  `shell_input_button_pressed()`, which explicitly catches a press+release that
  lands inside one poll. Contrast: the volume keys on the vendor node
  (`KEY_VOLUMEUP/DOWN`) *do* report sustained presses (288–321 ms measured), so
  the shell can see holds when a device produces them.
- **"button 0x… pressed" log lines come from the query, not the event.** They
  are emitted inside `shell_input_button_pressed()`, so a button that no screen
  queries logs nothing. Absence of a line does not prove the button never fired.
  Log explicitly in the action handler instead (the screenshot path now logs
  `screenshot requested (ARMOURY CRATE|COMMAND)`).
- Reserved vendor keys are opened by the shell as *roles* (`asus`, `vendor`,
  `power`); the raw-debug lines print the role, not the device name.

## Boot / init

- **Kernel config drops dependencies silently.** The Ally kernel once lost its
  entire USB stack because a Kconfig edit dropped deps (fixed `9aed2a4`). When
  editing `linux.config`, re-check the generated `.config` and test USB/NIC in QEMU.
- **ESP can appear late on NVMe.** `playos-init` retries ESP find+mount for late
  partition registration (`f82a887`). Don't assume the ESP exists at first scan.
- **`/data` can mount the wrong partition.** Prefer the removable USB stick over
  the internal NVMe `playos-data` during install/first boot (`974e3ca`,
  `e81aaae`).
- **Rootfs is read-only squashfs.** Persistent state goes under `/data` ONLY.
  Runtime `mkdir` under `/` fails; create bind-mount targets in the overlay
  (e.g. `/root/.ssh` for Dropbear).
- **`/tmp` must be tmpfs** on the squashfs root (`9fb5a49`), or tools that
  mktemp in `/tmp` fail.

## Compositor / graphics

- Compositor builds need wlroots 0.20 from `/opt/playos-deps`:
  `PKG_CONFIG_PATH=/opt/playos-deps/lib/x86_64-linux-gnu/pkgconfig:/opt/playos-deps/share/pkgconfig`.
- wlroots pc files pull `wayland-protocols >= 1.47` — include
  `/opt/playos-deps/share/pkgconfig` in the path or configure fails.
- Black-screen root causes were render loop + initial configure ordering
  (`99309817`). The compositor readiness file is `/run/playos/compositor-ready`.

## Audio

- **ALSA card routing:** Ally card 0 is the GPU's HDMI audio; the internal
  speakers are card 1 (ALC294 → CS35L41). `etc/asound.conf` sets
  `defaults.pcm.card 1` / `defaults.ctl.card 1`. Games need `/dev/snd` in the
  Landlock allowlist and `playos-game` in the `audio` group or audio breaks.
- Shell retries audio init until card 1 registers late (`31bc6ed`).
- **One PCM owner at a time — and the shell currently never lets go.** The shell
  opens the default playback PCM at startup and keeps it:
  `/proc/<shell>/fd/25 -> /dev/snd/pcmC1D0p` with
  `card1/pcm0p/sub0/status: state: RUNNING`. A game that then calls
  `InitAudioDevice()` fails with
  `ALSA lib pcm_dmix.c:1000:(snd_pcm_dmix_open) [error.pcm] unable to open slave`
  → `WARNING: AUDIO: Failed to initialize playback device`, and the game runs
  silent (Invaders logs `[audio] no audio device — running silent`). It is
  **intermittent**: in an earlier boot the shell's own audio init had failed
  (`snd_mixer_attach(default) failed` in the overlay log), the PCM was free, and
  the same game logged `procedural SFX ready` and played fine. Per ADR-0007 the
  shell must release the PCM when a game goes foreground and re-acquire on exit.
  So when debugging "no game audio", check **who holds `pcmC1D0p`** first — the
  game logging nothing is a symptom, not the cause.

## Security / sandbox (Sprint 12)

- **Sandbox order is fixed:** `no_new_privs → Landlock → setuid → seccomp → exec`.
  Landlock's `restrict_self` needs `no_new_privs`; seccomp denies `setuid`, so
  the credential drop must happen before the filter installs.
- **Landlock `add_rule` on a missing path fails** (`ENOENT`) — per-game
  saves/cache dirs are pre-created (`0700`, chowned 1001) before rules are added.
  An `allowed_access = 0` rule is rejected (`ENOMSG`); default-deny is the
  mechanism, not explicit deny rules.
- **`/etc/group` overlay replaces the generated file.** Adding a group
  membership means editing `board/common/rootfs-overlay/etc/group`, not just
  the users table.
- **seccomp install can be blocked by an outer sandbox.** On this dev host
  `prctl(PR_SET_SECCOMP)` returns `EACCES`, so `test_security` skips that check
  (exit 30). The filter must be validated in QEMU/on-device.
- **Games are dynamically linked** (musl + shared libraylib/libplayos) — the
  Landlock allowlist must include `/lib` and `/usr/lib` (read+execute) or every
  game exec fails on the interpreter. This is why a full seccomp *allowlist*
  was deferred in favor of a deny-list.
- **Ed25519 verify uses point `pack`, not raw field `pack25519`** for the R
  comparison — mixing them up silently fails verification.

## IPC / distro

- **The component repos are symlinked, not copied.** `playos-refdistro/src/playos-*`
  are symlinks to the sibling repos (`../../playos-init`, etc.), so
  `src/playos-init/ipc/ipc.h` IS `playos-init/ipc/ipc.h` — there is only one
  file. Edit and commit in the component repo; refdistro shows nothing for it
  (`src/playos-*` is gitignored). Exceptions that are real in-repo dirs:
  `src/playos-installer`, `src/playos-overlay`, `src/playos-raylib`.
- **Local packages are only rsync'd on first Buildroot configure.** The
  Makefile `dirclean`s all `playos-*` packages before each build so `src/`
  edits are picked up. Don't bypass that.
- **`versions.lock` pins are consumed by the build.** After merging a component
  change that the image should use, bump the pin
  (edit `versions.lock` directly — no helper script yet).
- Control socket is `0660 root:1000`; peer check accepts gid 1000 **or uid 0**.
  The shell/overlay currently pass via uid 0 (they still run as root).

## Tests

- `test_security` runs Landlock/seccomp in forked children and reports via exit
  codes — don't call `assert()` in those children.
- Host `ctest` for the compositor currently has **no tests**; it only proves
  compilation.
- Spec repo builds with `mdbook build`; a broken link or table fails it.

## Hardware validation is its own gate

QEMU/host passing ≠ done. Sprints 11.5–13.7 are on-device validated on the ROG
Ally and ZenBook. The remaining hardware gate is Sprint 14: the 19-criterion
MVP smoke test, the performance baseline, and SimpleDRM/low-graphics recovery
validation. See [`09-next-steps.md`](09-next-steps.md).

## When two callers share code, every implicit assumption surfaces (S14.5)

Extracting the install engine into `libplayos-install` so the standalone installer
and the screen-less `playos-install-worker` could share it was right, but the
second front-end exposed four things the first one had always quietly done for it.
Three are the same shape, and none appeared in a build, a code review or a QEMU
run - only on the device:

- **Name-based lookup is ambiguous when both media share partition names.**
  `playos-a` exists on the stick *and* on the internal disk (where it is a raw
  squashfs slot, not the ext2 payload). Init's label lookup found the internal one
  and `mount` failed with `EINVAL`, so the worker never started. The shell already
  found the payload by *contents*, so it now passes that device through the IPC.
- **Device paths are not normalised.** The shell sends `/dev/nvme0n1`; the step
  helpers add `/dev/` themselves, giving `/dev//dev/nvme0n1`. Normalise once, at
  the shared boundary, and check both spellings.
- **Mountpoints are preconditions.** `efi.c` mounts at `/mnt/efi` without creating
  it; only the standalone installer's `main()` did that. The error reads
  `mount /dev/nvme0n1p1: No such file or directory`, which looks like a missing
  *device* but is a missing *directory* - check `ls -ld` on the mountpoint before
  believing a device error.
- **Replacing a call is not replacing its consequences.** The shell's hold branch
  ended with `current_screen = SCREEN_SETTINGS`, correct when the install was a
  handoff to a separate app. The new flow kept drawing its cards - on a screen the
  user was not looking at, so a complete, successful install looked like nothing
  happened. When you replace a mechanism, grep for what the *old* mechanism was
  responsible for, not just the call you removed.

## Emulator / headless QEMU (S15-T7)

- **"The compositor is running" is not "the session can accept a game."**
  Autostart fired as soon as `compositor_state == COMPOSITOR_RUNNING`, but the
  compositor's *control connection* and the shell's `ShellReady` listener register
  a fraction of a second later. `SetExpectedGame` was dropped
  (`no compositor connection; cannot send SetExpectedGame`) and `GameStarted` had
  nowhere to go, so the game ran and rendered raylib into a surface the compositor
  never classified (`playos-compositor: unclaimed surface`) and `fps ... game=0`.
  Gate any programmatic launch on `compositor_conn_fd >= 0 && shell_listener_fd >= 0`.
- **The guest's log files are empty in a killed-QEMU disk image.** QEMU is stopped
  with SIGTERM, so `/data/log/*` inodes still show size 0: the writes (and the
  init.log fd) are in the ext4 journal. Replay it on a *copy* first
  (`cp` + `e2fsck -fy`, then `debugfs -R "cat /log/..."`). `debugfs` also pads
  its output with NULs, which makes `grep` call the file binary — `tr -d '\0'`
  or `grep -a`.
- **Don't give QEMU two GPUs and expect the compositor to pick the good one.**
  With the default VGA plus `-device virtio-gpu-pci`, wlroots selected
  `card1` (bochs-drm) and every atomic commit failed
  (`connector Virtual-2: Atomic commit failed: Out of memory`) while `fps shell=N`
  kept counting composited frames. `-vga none` + virtio-gpu removed the errors.
- **QEMU's kernel had no evdev at all.** `board/qemu-x86_64/linux.config` had
  `INPUT_KEYBOARD`/`ATKBD` but `# CONFIG_INPUT_EVDEV is not set`, so the guest
  exposed no `/dev/input/event*` and a game could never receive input. Enable
  `INPUT_EVDEV` (and `VIRTIO_INPUT` for `virtio-keyboard` / `-object input-linux`
  pass-through) — the game's input backend then finds the devices and honestly
  reports "no controller" for a keyboard rather than silently reading nothing.
- **The game's own stderr is the best render probe.** raylib logs
  `Platform backend: PLAYOS (Wayland + EGL/GLES2)` and
  `DISPLAY: Device initialized successfully`; if the compositor log then shows
  `game surface added to scene (role 3)` + `game=N`, the device artifact really
  rendered. A blank compositor log with those lines present means a policy/role
  problem, not a graphics problem.

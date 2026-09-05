# hyprland-ubuntu-parallels

Notes and configuration reference for Hyprland on Ubuntu 24.04 (arm64) inside a Parallels VM on macOS.

This is the setup I actually use on my personal work machine — not a generic
guide. Details like screen resolution or the host display are specific to my
hardware and are called out as such where it matters.

This repository **does not contain copies of the config files** (to avoid a
second source of truth that goes stale): it documents the *why* behind the
choices made in the real configuration, which lives in `~/.config/hypr/` and
the other `~/.config/*` dirs on the VM. Every note is anchored to a specific
file, line, or verifiable command.

## Environment

- Host: macOS, Parallels Desktop
- Guest: Ubuntu 24.04.4 LTS, arm64 (aarch64), 2 vCPU
- GPU: Virtio 1.0 GPU (virtio-gpu, `virtio-pci` driver), no native acceleration
- Session: Wayland, started by GDM (`Hyprland` entry in `/usr/share/wayland-sessions/`)
- Hyprland 0.56.2, installed from the `cppiber/hyprland` PPA (not in Ubuntu 24.04 repos)

## Index

- [docs/setup.md](docs/setup.md) — installation, PPA, package versions, known state
- [docs/graphics.md](docs/graphics.md) — virtual GPU, resolution/EDID, cursor, performance
- [docs/shortcuts.md](docs/shortcuts.md) — keybindings, why `mainMod` is ALT and not SUPER
- [docs/clipboard.md](docs/clipboard.md) — host↔VM copy/paste and in-session clipboard history
- [docs/waybar.md](docs/waybar.md) — top bar modules, layout, and styling
- [docs/wallpaper.md](docs/wallpaper.md) — hyprpaper → swww migration, build-from-source install, curated wallpaper set
- [docs/config-gotchas.md](docs/config-gotchas.md) — config syntax that changed across versions and silently broke something
- [docs/dev-tools.md](docs/dev-tools.md) — VS Code (native Wayland via the Electron ozone hint), Database Client extension over SQLTools, local Redis, client-only PostgreSQL

All notes reference specific package versions: if you upgrade a component and
something stops matching, that's the first thing to check.

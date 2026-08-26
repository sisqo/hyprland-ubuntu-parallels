# Setup

## Environment

- Host: macOS + Parallels Desktop
- Guest: Ubuntu 24.04.4 LTS, arm64 (aarch64), 2 vCPU, hostname `user-virtual-machine`
- GPU: `Red Hat, Inc. Virtio 1.0 GPU`, `virtio-pci` driver (see `lspci | grep -i vga`)
- Login: GDM → Wayland session "Hyprland" (`/usr/share/wayland-sessions/hyprland.desktop`,
  `Exec=/usr/bin/start-hyprland`)

## Installing Hyprland

Ubuntu 24.04 doesn't ship a recent Hyprland in its official repos. It's
installed from a third-party PPA:

```
ppa:cppiber/hyprland
```

(`/etc/apt/sources.list.d/hyprland-ppa.sources`, key in
`/etc/apt/keyrings/cppiber-hyprland.asc`).

**waybar is the exception**: it comes from the Ubuntu repo (`noble`), not the
PPA. It's an older version (0.9.24) than what Hyprland expects to talk to —
see [config-gotchas.md](config-gotchas.md#waybar-and-hyprlands-ipc-socket)
for the problem this causes.

## Installed versions (checked 2026-08-26, `dpkg -l`)

| Package | Version | Source |
|---|---|---|
| hyprland | 0.56.2-1ppa1 | cppiber PPA |
| hypridle | 0.1.8-1ppa1 | cppiber PPA |
| hyprlock | 0.9.6-1ppa1 | cppiber PPA |
| hyprpaper | 0.8.4-1ppa2 | cppiber PPA |
| hyprpolkitagent | 0.1.3-1ppa3 | cppiber PPA |
| waybar | 0.9.24-1build3 | Ubuntu repo (noble) |
| mako-notifier | 1.8.0-2build2 | Ubuntu repo |
| rofi | 1.7.5-0.1build2 | Ubuntu repo |
| wofi | 1.4.1-1build2 | Ubuntu repo (installed, unused — see [shortcuts.md](shortcuts.md#launcher-and-clipboard-picker-rofi)) |
| wlogout | 1.1.1-3build2 | Ubuntu repo |
| waypaper | 2.8 | pipx (`~/.local/share/pipx/venvs/waypaper`), not apt — see [config-gotchas.md](config-gotchas.md#pathlocalbin-not-visible-to-bind--exec) |
| cliphist | 0.4.0-2ubuntu0.3 | Ubuntu repo |
| wl-clipboard | 2.2.1-1build1 | Ubuntu repo |
| grim | 1.4.0+ds-2build2 | Ubuntu repo |
| slurp | 1.5.0-1 | Ubuntu repo |
| foot | 1.16.2-2ubuntu0.1 | Ubuntu repo |
| nautilus | 1:46.4-0ubuntu0.2 | Ubuntu repo |

These versions are the baseline for every "in version X, Y happens" note in
the other docs. If you upgrade a package, re-check those notes before
trusting they still hold.

## Parallels Tools

`prlcc` (`/usr/bin/prlcc`) is the guest-side Parallels Tools daemon — shared
clipboard host↔VM, drag&drop, dynamic screen resize. It's started via
`exec-once` in `hyprland.conf`, not as a systemd service (there is no
`prlcc.service`). Clipboard details in [clipboard.md](clipboard.md); dynamic
resize (which doesn't work here) in [graphics.md](graphics.md).

## Known state / things to verify

Observations from reading the current config, not yet acted on:

- `~/.config/hypr/monitors.conf` and `workspaces.conf` exist but are **empty
  and not included** by any `source =` in `hyprland.conf`: they look like
  placeholders for a future config split, currently dead.
- `~/.config/hypr/hypridle.conf` has a 600s auto-lock listener **commented
  out** (`# on-timeout = loginctl lock-session`); the only active listener
  turns off DPMS at 900s. In practice the screen blanks after 15 minutes of
  inactivity but the session **does not lock automatically**.
- `bind = ALT, F4, killactive,` in `hyprland.conf` is redundant with
  `$mainMod, Q, killactive,` since `$mainMod` is already `ALT` — almost
  certainly a leftover from when `mainMod` was `SUPER` (see
  [shortcuts.md](shortcuts.md)).

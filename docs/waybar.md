# Top bar (waybar)

Config in `~/.config/waybar/config` (modules/layout, JSON) and
`~/.config/waybar/style.css` (styling). For why waybar is launched through a
wrapper script instead of directly, see
[config-gotchas.md](config-gotchas.md#waybar-and-hyprlands-ipc-socket).

## Layout

30px bar, docked to the top layer.

| Zone | Modules |
|---|---|
| Left | `hyprland/workspaces` (click to switch), `hyprland/submap` |
| Center | `hyprland/window` — focused window title, truncated at 60 chars, per-output |
| Right | `idle_inhibitor`, `cpu`, `memory`, `disk`, `pulseaudio`, `network`, `clock`, `tray`, `custom/power` |

## Notable module settings

- **idle_inhibitor**: toggles between `AWAKE`/`IDLE` text (no icon font
  assumed). Only inhibits *screen* idling — it doesn't affect the missing
  auto-lock listener noted in
  [setup.md](setup.md#known-state--things-to-verify).
- **memory** / **disk**: `warning`/`critical` color states at 70/90% (RAM)
  and 80/95% (disk usage on `/`).
- **pulseaudio**: click toggles mute via `wpctl set-mute @DEFAULT_AUDIO_SINK@ toggle`.
- **network**: shows `essid` + signal for Wi-Fi, `ipaddr` for ethernet,
  `offline` when disconnected.
- **custom/power**: click runs `wlogout -b 4` — same power menu as
  `mainMod`+X (see [shortcuts.md](shortcuts.md)).

## Style

Colors match the Tokyo Night palette used across the rest of the setup
(`hyprland.conf` borders, `mako` notifications, `hyprlock`): background
`#1a1b26` at 92% opacity, accent `#7aa2f7`, text `#c0caf5`/`#a9b1d6`, warning
`#e0af68`, error/critical `#f7768e`. Active workspace pill and hover states
use rounded corners (6px) against an otherwise square, borderless bar
(`border-radius: 0` on `*`).

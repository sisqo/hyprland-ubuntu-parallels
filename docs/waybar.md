# Top bar (waybar)

Config in `~/.config/waybar/config` (modules/layout, JSON) and
`~/.config/waybar/style.css` (styling). For why waybar is launched through a
wrapper script instead of directly, see
[config-gotchas.md](config-gotchas.md#waybar-and-hyprlands-ipc-socket).

## Layout

30px bar, docked to the top layer.

| Zone | Modules |
|---|---|
| Left | `custom/branding` (static "SisQo 1.0" label), `hyprland/workspaces` (click to switch), `hyprland/submap` |
| Center | `hyprland/window` — focused window title, truncated at 60 chars, per-output |
| Right | `idle_inhibitor`, `cpu`, `memory`, `disk`, `pulseaudio`, `network`, `clock`, `tray`, `custom/shortcuts`, `custom/power` |

## Notable module settings

- **custom/branding**: no `exec` — just a literal `format: "SisQo 1.0"`, so
  it's static text with no update interval. Styled in the same accent color
  as `clock` (`#7aa2f7`, bold). Applying a config change to this or any other
  module needs a full waybar restart, not just a reload signal — see
  [config-gotchas.md](config-gotchas.md#waybar-0924-doesnt-live-reload-on-sigusr2).
- **idle_inhibitor**: toggles between `AWAKE`/`IDLE` text (no icon font
  assumed). Only inhibits *screen* idling — it doesn't affect the missing
  auto-lock listener noted in
  [setup.md](setup.md#known-state--things-to-verify).
- **memory** / **disk**: `warning`/`critical` color states at 70/90% (RAM)
  and 80/95% (disk usage on `/`).
- **pulseaudio**: click toggles mute via `wpctl set-mute @DEFAULT_AUDIO_SINK@ toggle`.
- **network**: shows `essid` + signal for Wi-Fi, `ipaddr` for ethernet,
  `offline` when disconnected.
- **custom/shortcuts**: click runs `~/.config/hypr/scripts/shortcuts.sh`,
  which parses every `bind`/`bindm` line straight out of
  `~/.config/hypr/hyprland.conf` (substituting `$mainMod`) and shows them in
  a `rofi -dmenu` list. This is a live view generated from the actual config,
  not a copy — if it and [shortcuts.md](shortcuts.md) ever disagree, the
  script (and the config it reads) is the source of truth.
- **custom/power**: click runs `wlogout -b 4` — same power menu as
  `mainMod`+X (see [shortcuts.md](shortcuts.md)).

## Style

`font-family` is `"Ubuntu", "Noto Sans", "JetBrainsMono Nerd Font Mono",
sans-serif` — the Nerd Font is a fallback after the two text fonts (see
[setup.md](setup.md#fonts)). No module currently uses icon glyphs (e.g.
`idle_inhibitor` shows `AWAKE`/`IDLE` as plain text — see above), this just
makes glyph icons available to add later without a font install first.

Colors match the Tokyo Night palette used across the rest of the setup
(`hyprland.conf` borders, `mako` notifications, `hyprlock`): background
`#1a1b26` at 92% opacity, accent `#7aa2f7`, text `#c0caf5`/`#a9b1d6`, warning
`#e0af68`, error/critical `#f7768e`, `custom/shortcuts` in green `#9ece6a`
(hover `#b9f27c`). Active workspace pill and hover states
use rounded corners (6px) against an otherwise square, borderless bar
(`border-radius: 0` on `*`).

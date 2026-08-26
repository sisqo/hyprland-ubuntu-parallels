# Shortcuts

This table can drift from the live config over time. For the current,
authoritative list straight from `hyprland.conf`, click the keyboard icon
in the top bar (`custom/shortcuts` — see [waybar.md](waybar.md)), which runs
`~/.config/hypr/scripts/shortcuts.sh`.

## `mainMod` is ALT (Option), not SUPER

If you know Hyprland's usual cheatsheets, expect Super — this config
deliberately doesn't use it:

```
$mainMod = ALT
```

On this VM, Cmd (SUPER on a Mac keyboard) is macOS's own "system" modifier,
and Parallels intercepts/swallows Cmd+letter combos inconsistently: Cmd+Q got
translated into Ctrl+Q before reaching the guest, and Cmd+W / Cmd+B were
swallowed with nothing sent to the guest at all (verified with `wev`).
Option+letter always arrives clean and unmodified, so `mainMod` was moved to
`ALT` instead of patching exceptions key by key.

This also frees up SUPER for the keyboard-layout toggle below.

## Bindings (`~/.config/hypr/hyprland.conf`)

| Combo | Action |
|---|---|
| `mainMod` + Return | Open terminal (foot) |
| `mainMod` + Q | Close focused window |
| ALT + F4 | Close focused window (redundant with `mainMod`+Q since `mainMod` *is* ALT — see [setup.md](setup.md#known-state--things-to-verify)) |
| `mainMod` + Shift + M | Exit the whole session (extra Shift on purpose, so it can't fire by accident) |
| `mainMod` + E | Open Nautilus (file manager) |
| `mainMod` + V | Toggle floating |
| `mainMod` + R | App launcher (rofi, `drun` mode) |
| `mainMod` + P | Pseudotile |
| `mainMod` + J | Toggle split direction |
| `mainMod` + F | Fullscreen |
| `mainMod` + L | Lock screen (hyprlock) |
| `mainMod` + X | Power menu (wlogout) |
| `mainMod` + Shift + V | Clipboard history picker (cliphist + rofi) — see [clipboard.md](clipboard.md) |
| `mainMod` + B | Wallpaper picker (waypaper, hyprpaper backend) — bound to the full path `/home/user/.local/bin/waypaper`, see [config-gotchas.md](config-gotchas.md#pathlocalbin-not-visible-to-bind--exec) |
| Print | Screenshot a region (grim+slurp) to clipboard |
| Shift + Print | Screenshot the full screen to clipboard |
| `mainMod` + arrows | Move focus between windows |
| `mainMod` + 1–9 | Switch to workspace 1–9 |
| `mainMod` + Shift + 1–5 | Move focused window to workspace 1–5 (only 1–5 are bound, not 6–9 — asymmetry in the source config, not a typo here) |
| `mainMod` + left-click drag | Move window |
| `mainMod` + right-click drag | Resize window |
| Super + Space | Toggle keyboard layout (it ↔ us) |

## Launcher and clipboard picker (rofi)

Both `mainMod`+R and `mainMod`+Shift+V go through `rofi`, configured in
`~/.config/rofi/config.rasi`:

```
configuration {
	modes: "drun,run,window";
	show-icons: true;
	icon-theme: "Adwaita";
	drun-display-format: "{name}";
	font: "sans 18";
}

@theme "/usr/share/rofi/themes/gruvbox-dark-soft.rasi"
```

The theme is one of rofi's built-in themes (`/usr/share/rofi/themes/`,
shipped by the `rofi` package itself) — no download needed, just the
`@theme` path.

`drun` mode (app launcher) and `-dmenu` mode (generic picker, used for the
cliphist list) are both covered by this one config. `wofi` is still
installed (`1.4.1-1build2`) but nothing in `hyprland.conf` calls it anymore —
it's an unused leftover from before the switch to rofi, not a fallback in
active use.

## Keyboard layout toggle: why Super+Space and not Alt+Shift

`input { kb_layout = it,us; kb_options = grp:win_space_toggle }`.

The obvious choice, `grp:alt_shift_toggle`, was getting stolen by something
below Hyprland on this VM before Hyprland's bind engine ever saw the keys —
confirmed because the layout still switched even in cases where the
`mainMod`+Shift bind itself didn't fire, meaning something intercepted
Alt+Shift earlier in the stack (the config attributes this to a replaced
`libxkbcommon` on this VM). Since SUPER is unused (`mainMod` is ALT), the
toggle was moved to Super+Space instead of chasing key-by-key exceptions.

## Terminal (foot) copy/paste

`foot` (`~/.config/foot/foot.ini` only overrides the font — currently
`JetBrainsMono Nerd Font Mono`, size 11, see [setup.md](setup.md#fonts) —
clipboard bindings are the package defaults from `/etc/xdg/foot/foot.ini`):

| Combo | Action |
|---|---|
| Ctrl+Shift+C | Copy selection to clipboard |
| Ctrl+Shift+V | Paste from clipboard |
| Shift+Insert | Paste primary selection |
| Middle-click | Paste primary selection |

This is separate from `mainMod`+Shift+V, which pastes from the
*cliphist history*, not the current clipboard/selection — see
[clipboard.md](clipboard.md).

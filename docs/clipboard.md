# Copy/paste

Two independent layers, easy to conflate because both use the word
"clipboard":

1. **Host↔guest**, macOS ↔ VM — handled by Parallels Tools.
2. **In-session history**, inside the Wayland session — handled by
   wl-clipboard + cliphist.

## Layer 1: host↔guest clipboard (Parallels Tools)

`prlcc` (`/usr/bin/prlcc`, started from the `hyprland.start` handler in `hyprland.lua`)
provides clipboard sync between macOS and the VM, plus drag&drop and (in
theory) dynamic resize — the resize part doesn't actually work here, see
[graphics.md](graphics.md#edid-disabled--no-automatic-dynamic-resize).

### The "Parallels Shared Clipboard" ghost window

`prlcc` briefly remaps a window titled `Parallels Shared Clipboard` every
time the VM regains focus from the host. It has an empty window class and no
WM utility hints. Without a rule for it, dwindle tiles it like a normal
window, it then hides itself, and the real window on screen appears to
"jump" or reposition for an instant — confirmed with a real reproduction via
`foot -T`, not just theory.

Fix in `hyprland.lua`:

```lua
hl.window_rule({
    name  = "parallelsClipboardGhost",
    match = { title = "^(Parallels Shared Clipboard)$" },

    float   = true,
    size    = "1 1",
    move    = "0 0",
    no_anim = true,
})
```

The old `hyprland.conf` had a comment above this rule arguing for
`no_initial_focus` rather than `no_focus`, so the window wouldn't steal focus
without breaking clipboard sync. The rule itself never included
`no_initial_focus` (or any focus-related property), only `float`, `size`,
`move` and `no_anim`, and it works as is. That comment was dropped in the
Lua migration (see
[config-gotchas.md](config-gotchas.md#conf-hyprlang-config-deprecated-in-056-removed-in-057)).
Re-checked after the migration with `foot -T "Parallels Shared Clipboard"`:
`hyprctl clients -j` shows it `floating: true` at `[-9,-10]`, size `[19,21]`
(foot doesn't go down to 1x1, which is fine: the point is that dwindle
doesn't tile it).

Before the Lua migration this rule was a `windowrule { ... }` block. See
[config-gotchas.md](config-gotchas.md#windowrule-v2-vs-v3) for why the older
`windowrulev2 = RULE,selector` one-liner didn't work.

## Layer 2: in-session clipboard history (wl-clipboard + cliphist)

Two watchers feed a history database, started from the `hyprland.start`
handler:

```lua
hl.exec_cmd("wl-paste --type text --watch cliphist store")
hl.exec_cmd("wl-paste --type image --watch cliphist store")
```

`mainMod`+Shift+V opens a picker over that history and copies the chosen
entry back onto the live clipboard:

```lua
hl.bind(mainMod .. " + SHIFT + V", hl.dsp.exec_cmd("cliphist list | rofi -dmenu | cliphist decode | wl-copy"), { desc = "Cronologia appunti" })
```

(the picker is `rofi`, not `wofi` — see
[shortcuts.md](shortcuts.md#launcher-and-clipboard-picker-rofi))

This is a history browser, not "the" clipboard — it doesn't replace normal
copy/paste, it lets you reach back further than the single most recent
clipboard entry.

Screenshots also go through `wl-copy`:

```lua
hl.bind("Print", hl.dsp.exec_cmd('grim -g "$(slurp)" - | wl-copy'), { desc = "Screenshot area -> appunti" })
hl.bind("SHIFT + Print", hl.dsp.exec_cmd("grim - | wl-copy"), { desc = "Screenshot schermo -> appunti" })
```

`grim`/`slurp` work directly here without a screenshot portal.

## Per-app note

Individual apps still handle their own copy/paste on top of these two
layers — e.g. `foot` uses Ctrl+Shift+C/V (see
[shortcuts.md](shortcuts.md#terminal-foot-copypaste)), which is unrelated to
the `mainMod`+Shift+V history picker.

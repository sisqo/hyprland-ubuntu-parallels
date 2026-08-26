# Copy/paste

Two independent layers, easy to conflate because both use the word
"clipboard":

1. **Host↔guest**, macOS ↔ VM — handled by Parallels Tools.
2. **In-session history**, inside the Wayland session — handled by
   wl-clipboard + cliphist.

## Layer 1: host↔guest clipboard (Parallels Tools)

`prlcc` (`/usr/bin/prlcc`, started via `exec-once` in `hyprland.conf`)
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

Fix in `hyprland.conf`:

```
windowrule {
    name = parallelsClipboardGhost
    match:title = ^(Parallels Shared Clipboard)$
    float = 1
    size = 1 1
    move = 0 0
    no_anim = 1
}
```

The comment above this block in the live config explains the property choice
as `no_initial_focus` rather than `no_focus` — not stealing focus without
breaking clipboard sync. **Note:** the block as it actually stands does not
include `no_initial_focus` (or any focus-related property) — only `float`,
`size`, `move`, `no_anim`. Worth checking whether that line was dropped by
accident or turned out to be unnecessary in practice; documented here as
observed, not "fixed".

See [config-gotchas.md](config-gotchas.md#windowrule-v2-vs-v3) for why this
rule uses the block syntax (`windowrule { ... }`) instead of the older
`windowrulev2 = RULE,selector` one-liner.

## Layer 2: in-session clipboard history (wl-clipboard + cliphist)

Two watchers feed a history database, started via `exec-once`:

```
exec-once = wl-paste --type text --watch cliphist store
exec-once = wl-paste --type image --watch cliphist store
```

`mainMod`+Shift+V opens a picker over that history and copies the chosen
entry back onto the live clipboard:

```
bind = $mainMod SHIFT, V, exec, cliphist list | wofi --dmenu | cliphist decode | wl-copy
```

This is a history browser, not "the" clipboard — it doesn't replace normal
copy/paste, it lets you reach back further than the single most recent
clipboard entry.

Screenshots also go through `wl-copy`:

```
bind = , Print, exec, grim -g "$(slurp)" - | wl-copy
bind = SHIFT, Print, exec, grim - | wl-copy
```

`grim`/`slurp` work directly here without a screenshot portal.

## Per-app note

Individual apps still handle their own copy/paste on top of these two
layers — e.g. `foot` uses Ctrl+Shift+C/V (see
[shortcuts.md](shortcuts.md#terminal-foot-copypaste)), which is unrelated to
the `mainMod`+Shift+V history picker.

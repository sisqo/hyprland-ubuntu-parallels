# Config syntax gotchas

Things that changed between versions of the Hyprland ecosystem and either
failed silently or produced a confusing symptom. Each entry is pinned to a
version — see [setup.md](setup.md#installed-versions-checked-2026-08-26-dpkg--l)
for the current baseline.

## waybar and Hyprland's IPC socket

waybar 0.9.24 (the Ubuntu `noble` repo version — installed separately from
the Hyprland PPA, see [setup.md](setup.md#installing-hyprland)) still looks
for Hyprland's IPC socket at `/tmp/hypr`. Hyprland moved that socket to
`$XDG_RUNTIME_DIR/hypr/<instance signature>` back in 0.42. Without a fix,
waybar asserts and dies on startup.

Fixed with a wrapper script, `~/.local/bin/waybar-hypr`, that symlinks the
expected legacy path to the real socket before exec'ing waybar:

```sh
#!/bin/sh
mkdir -p /tmp/hypr
ln -sfn "$XDG_RUNTIME_DIR/hypr/$HYPRLAND_INSTANCE_SIGNATURE" \
        "/tmp/hypr/$HYPRLAND_INSTANCE_SIGNATURE"
exec waybar "$@"
```

`hyprland.conf` launches this wrapper instead of `waybar` directly:

```
exec-once = /home/user/.local/bin/waybar-hypr
```

If waybar ever gets upgraded past 0.9.24 (or replaced by a build that reads
`$XDG_RUNTIME_DIR` directly), this wrapper becomes unnecessary and can be
dropped.

## hyprpaper 0.8 syntax change

hyprpaper 0.8.4 (installed here) changed its config format:

- `preload = <path>` no longer exists.
- `wallpaper = <monitor>,<path>` is silently ignored — not an error, just a
  no-op.

Current syntax is a keyed block per wallpaper, and `path` also accepts a
directory:

```
splash = false
ipc = on

wallpaper {
    monitor =
    path = /usr/share/backgrounds/Fuji_san_by_amaral.png
    fit_mode = cover
}
```

`ipc = on` is needed for `hyprctl hyprpaper wallpaper ...` at runtime, which
is also what the wallpaper picker (`mainMod`+B → `waypaper --backend
hyprpaper`) relies on.

## windowrule v2 vs v3

In Hyprland 0.56.2, the old one-liner syntax:

```
windowrulev2 = float,title:^(Example)$
```

is a **hard config error**, not just deprecated — it gets discarded silently
by the parser (see `src/config/legacy/ConfigManager.cpp` and
`src/desktop/rule/windowRule/WindowRuleEffectContainer.cpp` in Hyprland's
source). A rule written this way simply doesn't apply, with no warning that's
easy to notice.

The current syntax is a keyed block, with `snake_case` property names that
differ from the old v1/v2 ones (e.g. `no_anim`, `no_initial_focus` instead of
`noanim`, `nofocus`):

```
windowrule {
    name = someRuleName
    match:title = ^(Example)$
    float = 1
}
```

See [clipboard.md](clipboard.md#the-parallels-shared-clipboard-ghost-window)
for the concrete rule this project uses this syntax for.

## Keyboard layout toggle intercepted before Hyprland sees it

Not a syntax issue, but the same "silently not doing what the docs imply"
shape: `grp:alt_shift_toggle` was being consumed by something lower in the
stack on this VM (config attributes it to a replaced `libxkbcommon`) before
Hyprland's own bind handling ever ran — confirmed because the layout still
switched on Alt+Shift even when the `mainMod`+Shift keybind itself didn't
fire. Worked around by using `grp:win_space_toggle` instead. Full context in
[shortcuts.md](shortcuts.md#keyboard-layout-toggle-why-superspace-and-not-altshift).

## `vfr` renamed to `debug:vfr`

As of Hyprland 0.56, the `vfr` option lives under `debug:vfr` and is already
on by default — noted in a comment in `misc {}` in `hyprland.conf` so nobody
adds a `vfr = 1` line thinking it's required. See
[graphics.md](graphics.md#vrr-and-vfr) — this is unrelated to the separate
`vrr = 0` setting on the same block, which is about the (irrelevant on a
virtual monitor) variable refresh rate.

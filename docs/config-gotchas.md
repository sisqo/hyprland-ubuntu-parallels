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

## `~/.local/bin` not visible to `bind = ..., exec`

`mainMod`+B (wallpaper picker) silently did nothing when bound as plain
`exec, waypaper`. Root cause: the session is started by GDM
(`Service=gdm-autologin`, `Type=wayland` in `loginctl show-session`), which
never sources `~/.profile`/`~/.bashrc` — the files that normally put
`~/.local/bin` on `PATH`. `waypaper` is installed via `pipx`
(`~/.local/share/pipx/venvs/waypaper`, symlinked into `~/.local/bin`), not
apt, so it isn't on `PATH` in the environment Hyprland actually execs
commands in, even though it resolves fine from an interactive shell.

Fix: bind to the absolute path instead of relying on `PATH`:

```
bind = $mainMod, B, exec, /home/user/.local/bin/waypaper --folder /usr/share/backgrounds --backend hyprpaper
```

Same trap applies to any other pipx/pip-installed or otherwise
`~/.local/bin`-only binary invoked from a `bind`/`exec-once` line — it'll
work when typed in a terminal and do nothing when triggered from a keybind,
with no error visible anywhere obvious.

## rofi absolute width silently shrunk by a 0.625 factor

After enabling `xwayland.force_zero_scaling` (see
[graphics.md](graphics.md#xwayland-apps-render-blurry-and-undersized-at-scale-16)),
the obvious next step was to make rofi's launcher window bigger again by
setting an explicit `window { width: <N>px; }` in `~/.config/rofi/config.rasi`.
It didn't work as expected: every absolute width (`px`, or a bare number,
which rofi treats as `px`) came back **0.625× smaller than requested** —
e.g. asking for `2000px` measured out (via `hyprctl clients -j`) to 1250px,
and rofi's own compiled-in default width of `1280` (unstated anywhere in this
config) is why the window looked stuck at 800px physical pixels no matter
what `px` value was tried.

`%` values aren't affected — they're computed against the logical monitor
width (2560, i.e. `4096 / 1.6`) with no extra conversion, so `width: 45%`
gives exactly 1152px physical, as expected. Root cause not fully nailed down
(likely rofi's own DPI auto-detection interacting with
`force_zero_scaling`), but the workaround is simple: always use `%` for
`window { width: ...; }` in this rasi config, never `px` or a bare number.

Verified by isolating the effect with `rofi -no-config -theme-str
'window { width: <value>; }'` at several values, bypassing `config.rasi`
entirely, and reading the resulting window size back with
`hyprctl clients -j`.

## foot's `alpha` lives under `[colors]`, not `[main]`

Setting terminal transparency in `foot.ini` by adding `alpha=0.92` under
`[main]` (where `font=` already lives, so it seems like the natural spot)
fails outright rather than being ignored:

```
error: /home/user/.config/foot/foot.ini:3: [main].alpha: 0.92: not a valid option: alpha
```

foot 1.16.2 treats background opacity as a color property, not a general
`[main]` setting — it belongs under `[colors]`:

```
[main]
font=JetBrainsMono Nerd Font Mono:size=11

[colors]
alpha=0.92
```

At least this one fails loudly (`foot --check-config` catches it, and foot
itself refuses to start) instead of silently no-op'ing like most of the
other entries in this file — but the fix isn't obvious from the error text
alone since it doesn't say which section is correct.

## `apt remove` on the wallpaper package cascades into gdm3, gnome-shell, and ubuntu-desktop

While decluttering `/usr/share/backgrounds` (see
[wallpaper.md](wallpaper.md#wallpaper-images)), `sudo apt remove
ubuntu-wallpapers ubuntu-wallpapers-noble` looked like the "correct",
package-manager-tracked way to get rid of Ubuntu's stock wallpaper files —
cleaner than `rm`-ing files a package owns. apt's dependency resolver
disagreed: because `ubuntu-desktop`/`ubuntu-desktop-minimal` **directly
depend** on `ubuntu-wallpapers` (not just recommend it), removing the
wallpaper packages forced the removal of everything that depends on them,
transitively:

```
Remove: gnome-shell, ubuntu-wallpapers, ubuntu-wallpapers-noble,
ubuntu-desktop, gdm3, gnome-shell-extension-desktop-icons-ng,
gnome-shell-extension-appindicator, ubuntu-session,
gnome-shell-extension-manager, gnome-shell-extension-ubuntu-tiling-assistant,
gnome-shell-extension-ubuntu-dock, ubuntu-desktop-minimal
```

`apt remove` prints this list before acting, but on a `-y` run (or a
scroll-past) it's easy to miss that "remove two wallpaper packages" just
became "remove the display manager." Nothing broke *immediately* — `gdm3`
was still running as a live process (`systemctl status gdm.service` showed
`active (running)`) even after its package was gone, because removal
doesn't kill a running service — but the package itself showed `rc` (removed,
config-only) in `dpkg -l`, meaning the *binary was gone*. On this VM,
Hyprland is launched as a GDM Wayland session (see
[setup.md](setup.md#environment)), so the next reboot would have had no
display manager to start it, with no error until that reboot happened.

Recovery: reinstalling every package from the `Remove:` line above except
the two wallpaper ones (`apt install gnome-shell ubuntu-desktop gdm3 ...`)
brought `gdm3` back to `ii` (properly installed). Because
`ubuntu-desktop`/`ubuntu-desktop-minimal` still hard-depend on
`ubuntu-wallpapers`, apt pulled both wallpaper packages back in too as a
side effect — there's no way to keep those packages "installed" and their
files gone at the same time.

The actual fix for "declutter `/usr/share/backgrounds`" was to delete the
image files directly with `rm` (files inside a directory a package owns,
not the package's own files) instead of touching the package at all — the
approach originally passed over as less "correct":

```
sudo find /usr/share/backgrounds -mindepth 1 ! -name '<keep>' ... -exec rm -rf {} +
```

This leaves `dpkg -V ubuntu-wallpapers-noble` reporting missing files, which
is harmless (nothing depends on those specific files existing, only on the
package being *installed*) and easily reversed with
`apt install --reinstall` if ever needed. General lesson: before `apt
remove`-ing a package purely to get rid of files it owns, check `apt-cache
rdepends --installed <pkg>` (or read the `Remove:` line closely) — a
`Depends` (not `Recommends`) from a meta-package like `ubuntu-desktop` turns
a two-package removal into a cascade with no warning beyond that one line of
output.

## `vfr` renamed to `debug:vfr`

As of Hyprland 0.56, the `vfr` option lives under `debug:vfr` and is already
on by default — noted in a comment in `misc {}` in `hyprland.conf` so nobody
adds a `vfr = 1` line thinking it's required. See
[graphics.md](graphics.md#vrr-and-vfr) — this is unrelated to the separate
`vrr = 0` setting on the same block, which is about the (irrelevant on a
virtual monitor) variable refresh rate.

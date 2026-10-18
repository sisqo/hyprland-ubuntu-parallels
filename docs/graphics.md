# Graphics and troubleshooting

The GPU is virtual (virtio-gpu over virtio-pci, exposed via `virgl`), not a
passthrough of a real GPU. The choices around monitor, cursor, and effects in
`~/.config/hypr/hyprland.lua` are all direct consequences of that, not
aesthetic preferences.

## EDID disabled → no automatic dynamic resize

```
$ sudo dmesg | grep -i edid
[    0.381207] [drm] features: +virgl -edid -resource_blob -host_visible
```

(needs `sudo`, or `journalctl -k`: plain `dmesg` without privileges is empty
because of `kernel.dmesg_restrict`).

The virtio-gpu driver here exposes `-edid`: no EDID means the guest never
receives the "the Parallels window was resized to WxH" information from the
host, so **prlcc can't auto-adapt the resolution** even though it's running.

`preferred` as the mode would not help either: Hyprland resolves it from its
own mode list, which here starts at `1280x960` (the static default without
EDID), not the real screen resolution — an easy trap to fall back into if
the config is ever regenerated from scratch. The mode is picked from sysfs
instead (next section).

### Automatic switch: MacBook display vs Studio Display

EDID is off, but the host's display still reaches the guest in another way.
Every time Parallels changes state, a drm uevent with `HOTPLUG=1` fires, and
the **first mode** in `/sys/class/drm/card1-Virtual-1/modes` (the
"preferred" one) changes. This was checked with `udevadm monitor --kernel
--subsystem-match=drm` while the window was toggled (Hyprland 0.56.2,
2026-10-03), and the Studio Display row by reading the file on 2026-10-18:

| Parallels state                         | first mode   |
|-----------------------------------------|--------------|
| full screen on the MacBook Pro 16"      | `3456x2168`  |
| full screen on the Studio Display (5K)  | `5120x2880`  |
| windowed (any size)                     | `1280x960`   |

`1280x960` is a fixed default. It does not report the real window size, so
windowed resize still doesn't work. The full-screen value does report the
host display, though. `5120x2880` is not in Hyprland's `availableModes`
(`hyprctl monitors all`, whose list is read once and goes stale), but
`hl.monitor` accepts it anyway.

The choice is made in `~/.config/hypr/hyprland.lua` itself. Hyprland's Lua
has `io.open`, so the config reads that first line when it is parsed and
passes it straight to `hl.monitor` as the mode, always at scale 2:

```lua
local mode = preferred_mode()   -- first line of .../card1-Virtual-1/modes
if mode == nil or mode == "1280x960" then
    mode = "5120x2880"
end
hl.monitor({ output = "Virtual-1", mode = mode, position = "auto", scale = "2" })
```

Scale 2 matches macOS's default "looks like" size on both panels: 1728×1084
on the MacBook, 2560×1440 on the Studio Display. A display with another
preferred mode gets its native resolution at scale 2 too, with no config
change. When windowed (or if the sysfs file is missing), it falls back to
the Studio Display setup.

**What breaks with a fixed mode:** until 2026-10-18 the line was a static
`4096x2160@60` at scale 1.6, picked as the highest mode Hyprland listed.
`4096x2160` is about 17:9 and the Studio Display is 16:9, so Parallels
shrank the frame to fit the width: **black bands above and below**, with no
error anywhere.

`~/.config/hypr/scripts/monitor-auto.sh` is started from
`hl.on("hyprland.start")`. It listens with `udevadm monitor` and runs
`hyprctl reload` after each drm `change` event, unless the new preferred
mode is `1280x960` (windowed). In that case it keeps the current setup. It
waits 1 s after each event so that Hyprland has handled the hotplug before
the reload.

Applying the mode from the script with `hyprctl eval` is the rejected
alternative. It breaks on any config reload, including the automatic one
when `hyprland.lua` is saved: the static `hl.monitor` line comes back with
no hotplug event, so nothing switches it again. With the decision in the
config, every reload is correct.

(This setup was first documented on 2026-10-03, but it never reached the
live config: on 2026-10-18 `hyprland.lua` still had the static line and the
script didn't exist. Both were written then. If the bands come back, check
`pgrep -af monitor-auto` and `grep -n io.open ~/.config/hypr/hyprland.lua`.)

### Procedure to change resolution by hand

1. List the modes available on the virtual monitor:
   ```
   hyprctl monitors all
   ```
   (monitor name: `Virtual-1`. That list can be stale, see above:
   `cat /sys/class/drm/card1-Virtual-1/modes` is the current one.)
2. Test live without touching the file:
   ```
   hyprctl eval 'hl.monitor({ output = "Virtual-1", mode = "<WxH>@<Hz>", position = "auto", scale = "<scale>" })'
   ```
   (`hyprctl keyword monitor ...` no longer works under the Lua config: it
   answers `keyword can't work with non-legacy parsers. Use eval.`)
3. If it looks good, make it permanent in the mode choice in
   `~/.config/hypr/hyprland.lua` (e.g. a special case for a new display),
   then:
   ```
   hyprctl reload
   ```

## XWayland apps render blurry and undersized at scale 1.6

X11 apps that go through XWayland (not native Wayland — e.g. `rofi` 1.7.5,
which has no `rofi-wayland` fork in the Ubuntu `noble` repos) don't get the
compositor's fractional scale applied the way native Wayland clients do.
XWayland rendered them at the logical resolution (2560×1350, i.e. unscaled),
and Hyprland then upscaled that buffer 1.6× for compositing — the result was
blurry ("non sembra retina, è tutto sgranato") on top of already being
undersized.

Fixed in `hyprland.lua`:

```lua
hl.config({
    xwayland = {
        force_zero_scaling = true,
    },
})
```

This makes XWayland clients render 1:1 against physical pixels — sharp, but
now unaware of the 1.6 scale, so they come out visually smaller than before.
Each affected app's own config has to compensate for the lost scale
individually; see [shortcuts.md](shortcuts.md#launcher-and-clipboard-picker-rofi)
for how `rofi` does it, and
[config-gotchas.md](config-gotchas.md#rofi-absolute-width-silently-shrunk-by-a-0625-factor)
for a DPI quirk that trips up the obvious fix.

Those compensations were tuned at scale 1.6 on `4096x2160`. The scale is now
2 (see [above](#automatic-switch-macbook-display-vs-studio-display)): `%`
widths are still right, but `pt` fonts in rofi come out 20% smaller on the
Studio Display (`4096 / 5120`) and larger on the MacBook. Not re-tuned as of
2026-10-18.

`force_zero_scaling` is a session-wide XWayland setting, not per-app. As of
2026-08-26, no other app on this VM uses XWayland (Chrome and `foot` both run
native Wayland — checked with `hyprctl clients -j | grep xwayland`), so this
had no side effects here, but any future X11-only app would need the same
size compensation.

## Cursor disappearing or flickering

virtio-gpu here doesn't expose a hardware cursor plane
(`hyprctl monitors all` → `hardwareCursorsInUse: false`). Without explicitly
disabling hardware cursors, the pointer disappears or flickers. Fix in
`hyprland.lua`, inside `hl.config({...})`:

```lua
cursor = {
    no_hardware_cursors = true,
    enable_hyprcursor   = true,
},
```

## Blur and shadows disabled on purpose

The virtual GPU tops out at OpenGL ES 3.0 (via virgl) with 2 vCPU behind it.
Window blur and shadows are expensive on this stack and are explicitly kept
off in `decoration = { blur = { enabled = false }, shadow = { enabled = false } }`,
and likewise `blur_passes = 0` on the background in `hyprlock.conf`. They're
not "forgotten", they should only be turned back on if this moves to a setup
with a real GPU.

## VRR and `vfr`

`misc = {...}` has `vrr = 0`: variable refresh rate is meaningless on a virtual
monitor, so it's disabled explicitly. The comment above that line in the
config also notes that the `vfr` option (a different setting from `vrr`) was
renamed `debug:vfr` starting in 0.56 and is already enabled by default — so
there's no need to set it manually.

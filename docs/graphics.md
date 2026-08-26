# Graphics and troubleshooting

The GPU is virtual (virtio-gpu over virtio-pci, exposed via `virgl`), not a
passthrough of a real GPU. The choices around monitor, cursor, and effects in
`~/.config/hypr/hyprland.conf` are all direct consequences of that, not
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
The monitor line in `hyprland.conf` is therefore static:

```
monitor = Virtual-1,4096x2160@60,auto,1.6
```

`preferred` instead of an explicit mode would resolve to `1280x960` (the
static default without EDID), not the real screen resolution — an easy trap
to fall back into if the config is ever regenerated from scratch.

`scale = 1.6` on `4096x2160` gives a logical resolution of 2560×1350, close
to the 2560×1440 of a "Retina-ish" external monitor (Studio Display) plugged
into the host. It needs to be changed by hand if the host display changes.

### Procedure to change resolution by hand

1. List the modes available on the virtual monitor:
   ```
   hyprctl monitors all
   ```
   (monitor name: `Virtual-1`. Note: at the time this doc was written,
   `availableModes` also included `5120x2880@59.99Hz`, higher than
   `4096x2160` — higher than what the current config comment claims is the
   top available mode. Check `hyprctl monitors all` for what's actually best
   before blindly trusting the existing line.)
2. Test live without touching the file:
   ```
   hyprctl keyword monitor "Virtual-1,<WxH>@<Hz>,auto,<scale>"
   ```
3. If it looks good, make it permanent by editing the `monitor =` line in
   `~/.config/hypr/hyprland.conf`, then:
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

Fixed in `hyprland.conf`:

```
xwayland {
    force_zero_scaling = true
}
```

This makes XWayland clients render 1:1 against physical pixels — sharp, but
now unaware of the 1.6 scale, so they come out visually smaller than before.
Each affected app's own config has to compensate for the lost scale
individually; see [shortcuts.md](shortcuts.md#launcher-and-clipboard-picker-rofi)
for how `rofi` does it, and
[config-gotchas.md](config-gotchas.md#rofi-absolute-width-silently-shrunk-by-a-0625-factor)
for a DPI quirk that trips up the obvious fix.

`force_zero_scaling` is a session-wide XWayland setting, not per-app. As of
2026-08-26, no other app on this VM uses XWayland (Chrome and `foot` both run
native Wayland — checked with `hyprctl clients -j | grep xwayland`), so this
had no side effects here, but any future X11-only app would need the same
size compensation.

## Cursor disappearing or flickering

virtio-gpu here doesn't expose a hardware cursor plane
(`hyprctl monitors all` → `hardwareCursorsInUse: false`). Without explicitly
disabling hardware cursors, the pointer disappears or flickers. Fix in
`hyprland.conf`:

```
cursor {
    no_hardware_cursors = true
    enable_hyprcursor = true
}
```

## Blur and shadows disabled on purpose

The virtual GPU tops out at OpenGL ES 3.0 (via virgl) with 2 vCPU behind it.
Window blur and shadows are expensive on this stack and are explicitly kept
off in `decoration { blur { enabled = false } shadow { enabled = false } }`,
and likewise `blur_passes = 0` on the background in `hyprlock.conf`. They're
not "forgotten", they should only be turned back on if this moves to a setup
with a real GPU.

## VRR and `vfr`

`misc {}` has `vrr = 0`: variable refresh rate is meaningless on a virtual
monitor, so it's disabled explicitly. The comment above that line in the
config also notes that the `vfr` option (a different setting from `vrr`) was
renamed `debug:vfr` starting in 0.56 and is already enabled by default — so
there's no need to set it manually.

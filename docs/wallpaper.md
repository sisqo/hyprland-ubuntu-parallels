# Wallpaper: hyprpaper → swww

## Why swww instead of hyprpaper

hyprpaper is still installed (PPA package, see
[setup.md](setup.md#installed-versions-checked-2026-08-26-dpkg--l)) but is
no longer started. It only supports an instant, uncrossfaded wallpaper swap;
[swww](https://github.com/LGFae/swww) does smooth transitions and can change
the wallpaper at runtime without restarting the daemon, for about the same
CPU/memory cost — worth the trade on a 2 vCPU VM (see
[README.md](../README.md#environment)) precisely because it doesn't add a
heavier one.

`hyprpaper.conf` and its `wallpaper { }` block (see
[config-gotchas.md](config-gotchas.md#hyprpaper-08-syntax-change)) are
unused now but left in place.

## Install: built from source, not packaged

swww ships no prebuilt binaries and isn't on crates.io — `cargo install
swww` fails with "could not find `swww` in registry `crates-io`". The only
install path is building from the git repo, and there's no aarch64
prebuilt either way (checked the GitHub releases API: `v0.11.2` has zero
binary assets, source tarball only).

Ubuntu 24.04's own `rustc`/`cargo` (1.75.0) are also too old — swww
`v0.11.2`'s `Cargo.toml` pins `rust-version = "1.89.0"`. Installed
[rustup](https://rustup.rs) instead (`~/.cargo`, `~/.rustup`), which pulled
current stable (1.98.0 as of this install).

Build deps not present by default: `liblz4-dev` (frame compression for
animated wallpapers) and `pkg-config`. Without `liblz4-dev`, the build fails
in the `common` crate with:

```
The system library `liblz4` required by crate `common` was not found.
```

The repo has two separate binaries (`swww` the client, `swww-daemon`), and
`cargo install --git <url> swww` alone errors with "multiple packages with
binaries found" — both need to be named explicitly:

```
cargo install --git https://github.com/LGFae/swww --tag v0.11.2 --locked swww swww-daemon
```

Installed to `~/.cargo/bin/swww` and `~/.cargo/bin/swww-daemon`.

## Daemon startup race on launch

`swww img` talks to `swww-daemon` over a socket named after the current
Wayland display (see `common/src/ipc/socket.rs` in the swww source) that
doesn't exist until the daemon has finished starting. Hyprland fires every
`exec-once` line concurrently with no ordering guarantee, so a naive

```
exec-once = swww-daemon
exec-once = swww img /path/to/wallpaper.png
```

races: the `img` call can reach the socket before the daemon has created
it, and fails silently (no visible error, wallpaper just never gets set).
swww's own client has no retry/wait logic (checked — no polling or
`ConnectionRefused` handling in `client/src/`).

Worked around with a wrapper script,
`~/.config/hypr/scripts/swww-init.sh`, that starts the daemon then polls
`swww query` (which only succeeds once the socket exists) before setting
the wallpaper:

```sh
#!/usr/bin/env bash
set -euo pipefail

wallpaper="${1:-/usr/share/backgrounds/tokyo-night-cafe-at-night.png}"

/home/user/.cargo/bin/swww-daemon &

for _ in $(seq 1 50); do
	/home/user/.cargo/bin/swww query &>/dev/null && break
	sleep 0.1
done

/home/user/.cargo/bin/swww img "$wallpaper" --transition-type fade
```

`hyprland.conf` launches this instead of `hyprpaper`:

```
exec-once = /home/user/.config/hypr/scripts/swww-init.sh
```

The default wallpaper argument doubles as the boot-time default; it's kept
in sync by hand with whatever `waypaper`'s own last-picked wallpaper is
(`~/.config/waypaper/config.ini`, `wallpaper =`) — the two aren't linked
automatically, so a wallpaper picked interactively survives a reboot only if
this script's default is also updated.

## Same `~/.local/bin` PATH trap as waypaper itself, one layer deeper

See
[config-gotchas.md](config-gotchas.md#localbin-not-visible-to-bind--exec)
for the base issue: Hyprland's `bind`/`exec-once` inherit a minimal system
PATH that doesn't include `~/.local/bin`, which is why the wallpaper picker
bind uses `waypaper`'s absolute path already.

Switching waypaper's backend to swww hits the *same* trap a level deeper:
`waypaper --backend swww` shells out to a bare `swww` command internally —
there's no config knob to give it an absolute path — so it's subject to
Hyprland's PATH regardless of how the picker itself was launched. `swww`
lives in `~/.cargo/bin`, which is no more on that PATH than `~/.local/bin`
is.

Fixed by symlinking both binaries into `/usr/local/bin` (already on
Hyprland's minimal PATH, confirmed in the base gotcha entry), rather than
trying to inject `~/.cargo/bin` via an `env = PATH,...` line in
`hyprland.conf`:

```
sudo ln -sf /home/user/.cargo/bin/swww /usr/local/bin/swww
sudo ln -sf /home/user/.cargo/bin/swww-daemon /usr/local/bin/swww-daemon
```

`~/.config/waypaper/config.ini` also has `backend = swww` now (was
`hyprpaper`), and the picker bind in `hyprland.conf` matches:

```
bind = $mainMod, B, exec, /home/user/.local/bin/waypaper --folder /usr/share/backgrounds --backend swww
```

## Wallpaper images

`/usr/share/backgrounds` no longer holds Ubuntu's stock wallpapers (see
[config-gotchas.md](config-gotchas.md#apt-remove-on-the-wallpaper-package-cascades-into-gdm3-gnome-shell-and-ubuntu-desktop)
for what removing the package that owned them actually did) — it's now a
hand-picked set of dark/night-themed images, matching the Tokyo Night
accent colors used elsewhere (`hyprland.conf` borders, waybar, mako — see
[waybar.md](waybar.md#style)). Not full clones of any source repo: each was
opened and checked individually before copying, skipping OS/DE-logo
wallpapers, images with a baked-in watermark or text overlay, low-resolution
files that would look soft at 4096×2160 (see
[graphics.md](graphics.md)), and identifiable copyrighted-character fan art.

| File | Source |
|---|---|
| `tokyo-night-cafe-at-night.png`, `tokyo-night-neon-sign.jpg` | [tokyo-night/wallpapers](https://github.com/tokyo-night/wallpapers) |
| `aesthetic-lofi-cat.png`, `aesthetic-lantern-village.png`, `aesthetic-dark-forest.png`, `aesthetic-night-skyline.png`, `aesthetic-moonlit-lake.png`, `aesthetic-arch-deer.png`, `aesthetic-noir-shacks.png`, `aesthetic-cyberpunk-market.png`, `aesthetic-red-sunset.png`, `aesthetic-rainy-street.jpg`, `aesthetic-pixel-pagoda.jpg`, `aesthetic-cabin-lake.png`, `aesthetic-japan-castle.png`, `aesthetic-sunset-deer.jpg`, `aesthetic-ink-wave.png` | [D3Ext/aesthetic-wallpapers](https://github.com/D3Ext/aesthetic-wallpapers) (`images/` folder — renamed from their original filenames for clarity) |

A three-wallpaper Catppuccin batch (from a reupload of the since-taken-down
`catppuccin/wallpapers`) was tried and then deleted again at the user's
request in the same session, so it isn't listed above — mentioned here only
so the gap in naming isn't mistaken for an accident.

These are plain files, not tracked by any package — `apt` doesn't know
about them and won't touch them on upgrade or autoremove.

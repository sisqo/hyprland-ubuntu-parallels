# Development tools

What's installed on the VM for day-to-day development, beyond the Hyprland
desktop itself. None of it is Hyprland-specific, but it all has to come back
after a rebuild, and a couple of the choices below were made *because* of
this VM (2 vCPU, 8 GB RAM, virtio-gpu at scale 1.6).

## VS Code (1.134.0, Microsoft apt repo)

`code` comes from Microsoft's own repo, not snap:
`/etc/apt/sources.list.d/vscode.list` →
`https://packages.microsoft.com/repos/code stable main` (arm64), key
`/usr/share/keyrings/microsoft-archive-keyring.gpg`.

It runs as a native Wayland client — `hyprctl clients -j` reports
`"xwayland": false` for class `code` — because `hyprland.conf` exports, in
its `### Ambiente` block (line 47):

```
env = ELECTRON_OZONE_PLATFORM_HINT,auto
```

That one line covers every Electron app started from the session (Claude
Desktop 1.30096.1, installed as a `.deb`, included). Without it Electron
falls back to X11, i.e. Xwayland, and gets the blurry/undersized rendering
described in
[graphics.md](graphics.md#xwayland-apps-render-blurry-and-undersized-at-scale-16).
No per-app flags file (`~/.config/code-flags.conf` or similar) is needed —
worth knowing before installing any other Electron app.

## Database client: the "Database Client" extension, not a standalone GUI

Extensions installed (`code --list-extensions --show-versions`, checked
2026-09-05):

| Extension | Version | Role |
|---|---|---|
| `cweijan.vscode-database-client2` | 9.0.2 | the one in use: data grid with inline editing, filters, export, ER diagram; PostgreSQL and Redis among the supported engines |
| `cweijan.dbclient-jdbc` | 1.4.6 | pulled in automatically by the above |
| `mtxr.sqltools` + `mtxr.sqltools-driver-pg` | 0.28.6 / 0.5.8 | first attempt, superseded: it's a query editor only, with no way to browse or edit data visually. Still installed, safe to remove |

Why an extension rather than a proper database GUI inside the VM: every
alternative costs resources this VM doesn't have to spare. DBeaver is
Java/SWT and heavy next to VS Code plus a browser on 2 vCPU; Beekeeper
Studio and DbGate are Electron (they would run native Wayland thanks to the
env above, so rendering isn't the problem — memory is). The other option
worth remembering is a native macOS client on the host (TablePlus, Postico)
pointed at the remote database: nothing to install in the VM at all, since
the database isn't in the VM anyway (see below).

**Connections are not in `settings.json`.** The extension keeps them in its
own storage inside VS Code's global state, not in
`~/.config/Code/User/settings.json`, so they can't be copied over or
versioned: after a rebuild they have to be re-created from the extension's
sidebar (`+` → engine → paste the connection URL in the URL field, which
fills the form → Connect → Save). Current targets:

- **PostgreSQL**: remote, managed on Neon, `sslmode=require`. The connection
  string lives in the consuming project's environment, not in any file on
  this VM — take it from there or from the Neon dashboard.
- **Redis**: local, `127.0.0.1:6379`, no password (see below).

## Redis (local, Ubuntu package)

`redis-server` 7.0.15 (`5:7.0.15-1ubuntu0.24.04.4`, Ubuntu repo), enabled
as a systemd service at boot. Listens on `127.0.0.1` and `::1` only, no
`requirepass`:

```
ss -ltnp | grep 6379      # 127.0.0.1:6379 and [::1]:6379
redis-cli ping            # PONG, no AUTH needed
```

Config is `/etc/redis/redis.conf`, not readable as the regular user, so
anything beyond the two checks above needs `sudo`.

## PostgreSQL: client only, no server

There is no `postgresql` server package on the VM and none is wanted: the
databases are remote (Neon), and the previous local MySQL 8.0 server was
purged on 2026-08-19 — 26 GB of data belonging to projects no longer
developed here, and ~400 MB of resident memory that mattered on this RAM
budget.

`psql` is 17.11 from the PGDG repo — `/etc/apt/sources.list.d/pgdg.list` →
`https://apt.postgresql.org/pub/repos/apt noble-pgdg main`,
`signed-by=/usr/share/postgresql-common/pgdg/apt.postgresql.org.asc` —
package `postgresql-client-17`. Ubuntu's own `postgresql-client-16` (16.15)
is also installed through the `postgresql-client` metapackage;
`postgresql-client-common`'s wrapper picks the newest, so `psql --version`
reports 17.11. The PGDG repo exists only for that client — nothing else on
the VM comes from it.

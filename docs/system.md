# System services

The stock Ubuntu 24.04 install ships services meant for bare metal or for
other hypervisors. This note covers what was removed or masked on this
Parallels VM and why, plus the local tuning of the few services that matter
here. Checked 2026-09-30, extended 2026-10-17.

## Services removed or masked

| Unit / package | Action | Why |
|---|---|---|
| snap `cups` 2.4.19 | `snap remove --purge cups` | Duplicate spooler. The apt `cups` 2.4.7 is the real one, and the snap only ran in proxy mode (`cups-proxyd` forwarding to `/run/cups/cups.sock`). Nothing used its `cups` slot: the `firefox` and `thunderbird` snaps print through `cups-control`, which goes to the system CUPS. |
| `open-vm-tools` 13.0.10 | `apt purge` (+ `autoremove`) | VMware guest tools. They never start on Parallels, because their VMware virtualization condition isn't met. |
| `modemmanager` 1.23.4 | `apt purge` | A VM has no modems (`mmcli -L`: "No modems were found"). |
| `multipathd.service` + `.socket` (multipath-tools 0.9.4) | `systemctl disable --now`, then `mask` | Root is a plain `/dev/sda2` ext4, with no multipath storage. The package is **not purged**, because `apt purge multipath-tools` also removes the `ubuntu-server` and `ubuntu-server-minimal` metapackages. |
| `gnome-remote-desktop.service` | `systemctl disable --now` | RDP/VNC server for GNOME. Unused: the VM is reached from the Parallels window. |
| `kerneloops.service`, `whoopsie.path`, `apport.service`, `apport-autoreport.{path,timer}`, `apport-forward.socket` | `systemctl disable --now` | Crash reporting to Ubuntu. Nobody reads those reports for this VM. Disabling only `apport.service` isn't enough: the path, timer and socket units re-trigger it. |
| `switcheroo-control.service` | `systemctl disable --now` | Switches between two GPUs. The VM has one (virtio-gpu). |
| `fwupd-refresh.timer` + `fwupd.service` | timer `disable --now`, service `mask --now` | Firmware updates. A VM has no firmware to flash. `fwupd.service` is static and D-Bus-activated, so with only the timer disabled it still started (~40 MB RSS); hence the mask. |
| `openvpn.service` | `systemctl disable --now` | Umbrella unit for `/etc/openvpn/*`. Both `client/` and `server/` are empty and NetworkManager has no VPN connection. |
| `qemu-kvm.service` | `systemctl disable --now` | See below: it did nothing here. |
| `sssd.service` + `sssd-*.socket` (7 sockets) | `systemctl disable --now` | Directory/LDAP login. Not used (local accounts only); the service was already inactive, but its sockets were listening. |
| `lxd-installer.socket` | `systemctl disable --now` | Installs the LXD snap on first `lxc` call. Not wanted. |

To revert any of these: `sudo systemctl unmask <unit>` (fwupd only) and
`sudo systemctl enable --now <unit>`. No package was removed for them.

Leftover config of removed packages (`dpkg -l` state `rc`, 65 packages: 57 from 19 old
`linux-image`/`linux-modules[-extra]` 6.8.0-* kernels, plus `systemd-timesyncd`,
`qrtr-tools`, `protection-domain-mapper`, `docutils-common`, `gtk-doc-tools`,
`libdbus-1-dev`, `libjs-jquery`, `libmecab2`) was purged on 2026-10-17 with
`dpkg --purge`.

The apt CUPS stays because printing works through it. Parallels shares the
Mac's printer: `prlshprint` listens on `127.0.0.1:30631`, and the queue in
`/etc/cups/printers.conf` points at it
(`DeviceURI ipp://localhost:30631/printers/...`).

**cloud-init** is left alone on purpose: it is already disabled by the
marker file `/etc/cloud/cloud-init.disabled` (`cloud-init status` →
`disabled`).

**`qemu-kvm.service`** was first left alone too (2026-09-30), then disabled
on 2026-10-17 for tidiness. It is a oneshot: with `KSM_ENABLED=AUTO` in
`/etc/default/qemu-kvm` it leaves `/sys/kernel/mm/ksm/run` at `0`, and the
guest has no `/dev/kvm`, so it never had anything to do.

The notification daemon also needed one unit masked, for a different
reason: see
[config-gotchas.md](config-gotchas.md#swaync-and-mako-both-claim-the-notification-bus-name).

## apt timers: not at boot

Stock `apt-daily.timer` (`OnCalendar=*-*-* 6,18:00`) and
`apt-daily-upgrade.timer` (`OnCalendar=*-*-* 6:00`) have `Persistent=true`.
The VM is usually off at those times, so every boot ran the missed jobs
right away: on the 2026-10-16 boot `systemd-analyze blame` showed
`apt-daily-upgrade` 25 s and `apt-daily` 11 s, running alongside login on 2
vCPU.

Both timers have the same drop-in,
`/etc/systemd/system/apt-daily{,-upgrade}.timer.d/override.conf`:

```
[Timer]
Persistent=false
OnBootSec=15min
```

`OnCalendar` stays as shipped. `OnBootSec=15min` keeps at least one run per
boot, so unattended security upgrades still happen on a VM that is rarely on
at 6:00, just not during login. Plain `Persistent=false` alone was rejected:
it could skip updates for days. Check with `systemctl list-timers 'apt-daily*'`.

### Boot time: plymouth isn't the bottleneck

`systemd-analyze` reports `graphical.target reached after ~23 s`, and the
critical chain blames `plymouth-quit-wait.service` (+21 s). That number is
misleading: in `journalctl -b -o short-monotonic`, `gdm.service` starts at
~7.1 s and the autologin session opens at ~7.3 s. The desktop is already
usable by then; `plymouth-quit-wait` just keeps `graphical.target` pending
until plymouth is told to quit. Removing `splash` from the kernel command line
would only change that number, so `GRUB_CMDLINE_LINUX_DEFAULT` stays at
`"quiet splash"`.

## Time sync: Parallels only

Parallels Tools (27.0.2.58673, `/usr/lib/parallels-tools/version`) runs
`prltimesync`, which keeps the guest clock in sync with the Mac.

`ntp` was also installed. It is a transitional package that pulls in
`ntpsec` 1.2.2, whereas Ubuntu's default would be `systemd-timesyncd`. The
remaining apt logs don't show when or why it was installed. The result was
two agents correcting the same clock. On every boot `ntpd` stepped the clock
by 0.19–0.49 s, in either direction:

```
journalctl | grep "CLOCK: time stepped"
```

On 2026-09-28 one of those negative steps made journald log
`Time jumped backwards, rotating`.

On 2026-09-30 `ntp`, `ntpsec` and `python3-ntp` were purged.
`prltimesync` was kept for two reasons: it follows the Mac's clock, which
macOS already keeps in sync, and ntpsec was the agent visibly stepping the
clock at every boot. Neither agent was seen fixing the big jumps: on
2026-09-15 nothing corrected the jump to 2121 before shutdown (see
[below](#clock-jumps-after-vcpu-stalls)). Don't install `systemd-timesyncd`
or `chrony` alongside `prltimesync`, or you'll have the same two agents
again.

Expected side effect: `timedatectl` reports
`System clock synchronized: no` / `NTP service: n/a`, because
`prltimesync` doesn't set the kernel's NTP-sync flag. To check the real
drift, compare `date -u` with an external clock:

```
curl -sI https://www.google.com | grep -i '^date:'
```

## Out-of-memory: earlyoom

The VM has 8 GB RAM and 8 GB of swap (`/swapfile`). Under a heavy dev load
it fails by thrashing: the desktop freezes long before the kernel OOM killer
fires.

`systemd-oomd` (Ubuntu's default, still active) doesn't cover this case.
`oomctl` shows no *Swap Monitored CGroups*, and the only pressure-monitored
cgroup is `user@1001.service`. Hyprland and everything launched from it live
in `session-2.scope` (`cat /proc/$(pgrep -x Hyprland)/cgroup`), outside that
cgroup.

`earlyoom` 1.7-2 (apt) is installed for this. It is configured in
`/etc/default/earlyoom`:

```
EARLYOOM_ARGS="-r 3600 -m 10 -s 50 --prefer ^(node|next-server) --avoid ^(Hyprland|Xwayland|start-hyprland|waybar|foot|gdm.*|gnome-keyring-d|pipewire.*|wireplumber|dbus-.*|systemd.*|prl.*|claude)$"
```

- **`-s 50`.** The stock free-swap threshold is 10%, which with 8 GB of swap
  means acting only after about 7 GB have been swapped out, well into the
  freeze. With `-s 50`, earlyoom sends SIGTERM when available RAM is ≤ 10%
  **and** free swap is ≤ 50%. It sends SIGKILL at half of both.
- **`--prefer ^(node|next-server)`.** A Next.js dev server is the usual
  runaway process: `next-server` once reached 2.2 GB RSS. earlyoom matches
  the 15-character `comm`, and the regex is anchored only at the start, so
  it matches whether `comm` is `node` or a truncated title such as
  `next-server (v1`.
- **`--avoid ...`** protects the compositor, the bar, the terminal, audio,
  D-Bus and the Parallels Tools daemons. It also protects Claude Code
  sessions (`comm` is `claude`, ~430–500 MB RSS each): killing one loses the
  whole session. Only the main process is protected; the shells and MCP
  servers it spawns are not.
- **No quotes around the regexes.** The unit runs
  `ExecStart=/usr/bin/earlyoom $EARLYOOM_ARGS`. systemd splits that value on
  whitespace but doesn't strip quotes, so quotes would become part of the
  regex. The regexes therefore must not contain spaces either.
- **No `-p`.** The unit only grants `CAP_KILL CAP_IPC_LOCK`, so the renice
  would just log an error.

The thresholds actually in use are printed at startup:
`journalctl -u earlyoom -b`.

## Clock jumps after vCPU stalls

Twice the guest stopped getting CPU from the host, with memory nowhere near
full:

| When | RAM / swap (sar) | What happened |
|---|---|---|
| 2026-09-15, ~22:05 | — | CPU starved for ~15 min. The guest clock then jumped to **2121-11-06**. |
| 2026-09-29, ~10:02–10:24 | 9% used / 0% | `systemd:1 blocked for more than 614 seconds` (in `synchronize_rcu`); snapd killed by its watchdog. |

The kernel signature was the same both times. RCU stall reports showed the
stalled CPU idle in `cpuidle_idle_call`, with
`rcu_preempt kthread timer wakeup didn't happen for N jiffies`. The idle
vCPU simply wasn't woken up. The cause is on the Mac/Parallels side and
isn't identified yet. Parallels Tools matches the host build
(27.0.2.58673), so it isn't a Tools/host mismatch.

The jump to 2121 left two kinds of damage behind:

- **The journal index broke.** The archived journal files whose tail
  timestamp was in 2121 threw off `journalctl --list-boots`, and
  `journalctl -b -1` failed with
  `No journal boot entry found from the specified boot offset (-1)`.
- **Files got future mtimes.** 290 files on `/` were written during the jump
  and got 2121 mtimes. They included polkit files from an unattended
  upgrade, the man-db index, the swcatalog cache, sysstat's `sa06`, and
  VS Code state. Mtime-based cleanup never removes these files, and freshness
  checks think those caches are up to date.

Cleanup done on 2026-09-30. It lost the logs of 2026-09-05..15.

```
# journal files whose tail is in the future (archived, not open)
for f in /var/log/journal/*/*.journal; do
  sudo journalctl --file "$f" --header | grep -q 'Tail realtime.*21[0-9][0-9]-' && echo "$f"
done
sudo rm <those files>

# everything else: reset future mtimes to now (pick a date after today)
sudo find / -xdev -newermt 2027-01-01 -exec touch -h -c {} +
```

To make sar history long enough to look back at incidents like these,
`HISTORY` in `/etc/sysstat/sysstat` went from `7` to `28`. That is the
maximum before sysstat switches to its `-D` file naming.

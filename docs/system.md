# System services

The stock Ubuntu 24.04 install ships services meant for bare metal or for
other hypervisors. This note covers what was removed or masked on this
Parallels VM and why, plus the local tuning of the few services that matter
here. Checked 2026-09-30.

## Services removed or masked

| Unit / package | Action | Why |
|---|---|---|
| snap `cups` 2.4.19 | `snap remove --purge cups` | Duplicate spooler. The apt `cups` 2.4.7 is the real one, and the snap only ran in proxy mode (`cups-proxyd` forwarding to `/run/cups/cups.sock`). Nothing used its `cups` slot: the `firefox` and `thunderbird` snaps print through `cups-control`, which goes to the system CUPS. |
| `open-vm-tools` 13.0.10 | `apt purge` (+ `autoremove`) | VMware guest tools. They never start on Parallels, because their VMware virtualization condition isn't met. |
| `modemmanager` 1.23.4 | `apt purge` | A VM has no modems (`mmcli -L`: "No modems were found"). |
| `multipathd.service` + `.socket` (multipath-tools 0.9.4) | `systemctl disable --now`, then `mask` | Root is a plain `/dev/sda2` ext4, with no multipath storage. The package is **not purged**, because `apt purge multipath-tools` also removes the `ubuntu-server` and `ubuntu-server-minimal` metapackages. |

The apt CUPS stays because printing works through it. Parallels shares the
Mac's printer: `prlshprint` listens on `127.0.0.1:30631`, and the queue in
`/etc/cups/printers.conf` points at it
(`DeviceURI ipp://localhost:30631/printers/...`).

Two services were left alone on purpose, since there's nothing to gain:

- **cloud-init** is already disabled by the marker file
  `/etc/cloud/cloud-init.disabled` (`cloud-init status` → `disabled`).
- **`qemu-kvm.service`** is a oneshot. With `KSM_ENABLED=AUTO` in
  `/etc/default/qemu-kvm` it leaves `/sys/kernel/mm/ksm/run` at `0`, and the
  guest has no `/dev/kvm`.

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

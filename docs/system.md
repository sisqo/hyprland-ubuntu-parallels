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

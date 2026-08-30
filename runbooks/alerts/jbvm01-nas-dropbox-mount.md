# NasDropboxMountAbsent / NasDropboxMountUnhealthy

| Alert | Severity | Condition |
|-------|----------|-----------|
| NasDropboxMountAbsent | warning | node_exporter on JBVM01 is up but exports no live CIFS mount at `/mnt/nas-media`, sustained 15m |
| NasDropboxMountUnhealthy | warning | `node_filesystem_device_error == 1` for the mount (statfs failing), sustained 10m |

**Signal**: JBVM01 mounts the NAS `Media` share at `/mnt/nas-media` as a write-only SMB dropbox (fstab + `x-systemd.automount`), and qBittorrent's download targets live under it. The mount is deliberately excluded from the generic `NodeFilesystem*` rules (its "free space" is a NAS-side user quota, not a local disk), so these two purpose-built alerts are the only coverage. **Absent** means the exporter no longer sees a mounted CIFS filesystem there; **Unhealthy** means the mount exists but `statfs` on it is erroring, the fingerprint of a wedged SMB client session.

## Why this matters

If the automount unit disappears (fstab edit, unit masked, drift after a rebuild), the path degrades to a plain local directory and downloads silently land on JBVM01's 50 GB root disk while everything looks fine — the exact incident shape that motivated the alert. With the automount in place a NAS outage fails loudly (I/O errors), which is the acceptable failure mode.

## Operator loop

1. On JBVM01: `findmnt -o FSTYPE,SOURCE,TARGET /mnt/nas-media`. Healthy shows **two** rows: an `autofs` trigger (`systemd-1`) and a `cifs` mount (`//<NAS>/Media`).
2. **Only the `autofs` row present**: normal immediately after boot until first access. Trigger the mount with a **write probe**, e.g. `touch /mnt/nas-media/_inbox/movies/.probe && rm` the same file. Never health-check with `ls` — the dropbox ACL is write-only and denies directory listing *by design*.
3. **Neither row present**: the automount is gone. Check `systemctl status 'mnt-nas\x2dmedia.automount'` and the fstab entry; restore, `systemctl daemon-reload`, `systemctl start` the automount unit. Audit `~/Downloads` and the qBittorrent save paths for files that landed locally in the gap, and move them into the dropbox.
4. **`cifs` row present but Unhealthy firing**: wedged client session. Confirm the NAS answers SMB from elsewhere first; if the NAS is fine, remount on the **client**: stop qBittorrent, `umount /mnt/nas-media`, then re-trigger with a write probe and restart qBittorrent.
5. If the NAS itself is down, this alert is collateral — work the NAS alert instead; this one clears on first write after the share returns.

## Learned the hard way

- A mount can look "connected" while every operation fails; remounting the client, not the server, clears a wedged SMB session.
- `x-systemd.automount` means "not mounted at boot" is healthy, not a fault — the mount appears on first access. That is why Absent carries a 15m hold.
- The alert pair splits "not there" from "there but broken" because the operator actions differ: restore the unit vs. bounce the session.

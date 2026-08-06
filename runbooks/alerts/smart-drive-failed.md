# SmartDriveStatusFailed

| Severity | Condition |
|----------|-----------|
| critical | `smartctl_device_smart_status == 0`, sustained 5m |

**Signal**: the drive's own overall SMART self-assessment has flipped to FAILED, meaning a manufacturer threshold has been crossed. This is the strongest pre-failure signal SMART produces; treat the drive as dying now.

## Triage

1. **Identify and map**: `smartctl -a /dev/<device>` on JBSRV01 for model and serial, then establish exactly what lives on it (which ZFS pool member, which LVM VG and VM disks). The serial matters: you will need it to pull the right drive from the chassis.
2. **Assess protection**: data on a parity-protected pool member survives replacement; data on an unprotected volume needs an immediate copy off the drive **before** any stress like a full backup or scrub (bulk reads can finish a dying drive).
3. **Replace at the next safe window**: for a ZFS member, `zpool replace` with the new drive and let the resilver run; verify with `zpool status -v` until it completes clean.
4. **After physical swap on a SAS/SATA HBA**: a warm-swapped drive may not enumerate; rescan the SCSI host (`echo "- - -" > /sys/class/scsi_host/hostN/scan`) instead of rebooting.

## While waiting for the replacement

- Do not run scrubs or full backups against the failing drive unless capturing data off it is the goal.
- Expect [smart-drive-warning.md](smart-drive-warning.md) attributes on the same drive; they're the same story. Silence duplicates for the replacement window rather than re-triaging.

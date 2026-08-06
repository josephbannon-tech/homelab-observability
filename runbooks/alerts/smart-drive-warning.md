# SmartDriveAttributesBad / SmartSsdLifeLeftLow

| Alert | Severity | Condition |
|-------|----------|-----------|
| SmartDriveAttributesBad | warning | non-zero raw Reallocated_Sector_Ct, Current_Pending_Sector, or Offline_Uncorrectable, 10m |
| SmartSsdLifeLeftLow | warning | SSD life-left normalised value below 5, sustained 1h |

**Signal**: smartctl_exporter on the hypervisor (JBSRV01 `:9633`), the only host that sees the physical drives (the NAS VM sees virtio devices). These are the plan-a-replacement alerts, deliberately warning-severity: they mean wear or media damage is real and trending, not that failure is imminent (that's [smart-drive-failed.md](smart-drive-failed.md)).

## Triage

1. **Confirm and capture the trend**: `smartctl -a /dev/<device>` on JBSRV01. Record the raw values; a static reallocated count from years ago is very different from one that moved this week.
2. **Map the drive to blast radius before anything else**: which pool or VM disk lives on it, and what parity protects it. A worn drive in single-parity RAIDZ1 means the pool is one *additional* failure from loss, which sets your replacement urgency.
3. **Pending vs reallocated**: `Current_Pending_Sector > 0` means unreadable-right-now sectors awaiting reallocation; treat it more urgently than a stable reallocated count, and scrub the pool to force the issue to resolve or surface.
4. **For SSD wear**: the alert fires on the normalised value crossing the threshold, i.e. approaching the manufacturer's floor. Source replacements now rather than at failure; when a mirrored/parity pair are the same age and model, plan to swap both in one window, since twins wear together.

## Notes

- These rules replaced an earlier alert on the smartctl exit-status bitfield, which latches historical error-log entries and therefore alerted forever on any past event. If a drive has an old error-log entry but clean attributes, it will (correctly) not alert.
- A drive can pass SMART entirely and still hang a SAS HBA under sustained load; absence of these alerts is not proof of drive health when chasing a hang (see [nas-burn.md](nas-burn.md)).

# Jbvm02MaintenanceStale

| Severity | Condition |
|----------|-----------|
| warning | weekly maintenance heartbeat older than 8 days, sustained 1h |

**Signal**: the weekly disk-hygiene job on JBVM02 (the development VM) writes `maintenance_last_success_timestamp_seconds` on success; it has gone quiet. This alert exists because that host once filled its root filesystem to 98% unnoticed; the maintenance job and this watchdog on it are the two layers that prevent a repeat.

## Triage

1. `systemctl status jbvm02-maintenance.timer` and the matching service unit on JBVM02: enabled, and when did it last fire?
2. If the timer fired but the heartbeat is stale, the script failed partway: `journalctl -u jbvm02-maintenance` for the error.
3. If someone rebuilt or renamed the job, remember the orphaned-heartbeat gotcha in [backup-heartbeat-stale.md](backup-heartbeat-stale.md): the old `.prom` file must be deleted, or this alert never clears.
4. While here, glance at `df -h /`: if maintenance has been dead a while, the disk-space alerts may be next; clearing the backlog now beats getting paged twice.

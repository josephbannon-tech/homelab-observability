# BackupHeartbeatStale / VzdumpHeartbeatStale

| Alert | Severity | Condition |
|-------|----------|-----------|
| BackupHeartbeatStale | warning | daily-cadence heartbeat older than 36h, sustained 1h |
| VzdumpHeartbeatStale | warning | weekly VM-backup heartbeat older than 8 days, sustained 1h |

**Signal**: every backup-shaped script writes `backup_last_success_timestamp_seconds{name="<job>"}` to its host's node_exporter textfile collector **on the success path only**. Silence is the failure signal, which is the whole design: a backup that stops running looks identical to one that runs and fails, and both page. Thresholds are cadence-aware (daily + 12h grace; weekly + 1d grace).

## Triage

1. **Which job, which host**: the alert's `name` and `hostname` labels identify both.
2. **Did it run at all?** Check the job's cron/timer on that host (`systemctl status <job>.timer` / crontab), then its log or `journalctl -t <name>`.
3. **Ran but failed?** The heartbeat only writes on success, so read the log for the actual error. Common shapes here: an unreachable target (NFS/SMB mount gone stale; check `mountpoint`), an SSH key rotated on one end, a tool missing after a rebuild.
4. **For the VM-backup variant**: on the hypervisor, check the latest vzdump log for the named guest, and confirm the backup job config still carries the hook script that writes heartbeats. Also confirm the backup storage is actually mounted.
5. **Ran, succeeded, no heartbeat?** That's pipeline breakage, not backup breakage: check the textfile collector directory is configured on that host's node_exporter, writable by the user the job runs as, and that `node_textfile_scrape_error` is 0.

## Textfile-collector gotchas (both silent)

- Two `.prom` files declaring conflicting `# TYPE` headers for the same metric name make node_exporter drop samples without logging an error.
- **Retiring or renaming a job orphans its old `.prom` file**, which node_exporter re-exports frozen forever, so the stale alert fires eternally for a job that no longer exists. Delete the file when retiring a job, and repoint dashboards/rules in the same change.

## Retention note

Backup-job retention on this platform is configured on the **storage** (`prune-backups`); an older job-level `maxfiles` setting is silently ignored on current Proxmox. If backups succeed but disk fills with old archives, that's where to look (`--dry-run` the prune to verify).

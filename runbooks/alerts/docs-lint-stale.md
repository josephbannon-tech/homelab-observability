# DocsLintStale

| Severity | Condition |
|----------|-----------|
| warning | last docs-lint run older than 36 hours, sustained 1h |

**Signal**: a nightly job on JBVM02 (the development VM) lints the private knowledge base for drift: follow-ups that never reached the queue, host docs whose `Last verified` date is older than their newest change-log entry, dead links between docs. It writes one gauge per finding kind (`homelab_docs_lint_findings`) and `homelab_docs_lint_last_run_timestamp_seconds`. The timestamp has stopped moving.

The finding gauges are a textfile metric, so they keep exporting their last values after the job dies. A dashboard showing zero findings then means nothing. This alert is the watchdog on that.

## Triage

1. `systemctl status docs-lint.timer docs-lint.service` on JBVM02: is the timer enabled, and when did it last fire?
2. If the timer fired but the timestamp is stale, the script failed before writing: `journalctl -u docs-lint -n 50`. A Python traceback usually means a doc was renamed or a new file broke a parser rule.
3. Run it by hand from the knowledge-base folder, `python3 scripts/docs-lint.py`, and read the output. A non-zero finding count is not a failure of the job; only a crash is.
4. If the job was renamed or moved, delete the old `.prom` file from the textfile collector directory. See the orphaned-heartbeat note in [backup-heartbeat-stale.md](backup-heartbeat-stale.md).

## Why 36 hours

The job runs once a night. 36 hours tolerates one missed run (a reboot across the scheduled time, with `Persistent=true` catching up) and fires on the second.

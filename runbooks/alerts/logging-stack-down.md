# LokiDown / PromtailDown

| Alert | Severity | Condition |
|-------|----------|-----------|
| LokiDown | warning | `up{job="loki"} == 0`, sustained 5m |
| PromtailDown | warning | `up{job="promtail"} == 0`, sustained 5m |

**Signal**: the logging pipeline's own health. LokiDown means ingestion has stopped estate-wide (dashboards go blank, agents buffer); PromtailDown covers the in-cluster agent shipping pod logs. The six external hosts' Promtail agents are **not** covered by these alerts; they are caught by the fleet check ([../verify-promtail-fleet.md](../verify-promtail-fleet.md)) or their host going dark entirely.

## LokiDown

1. `kubectl -n monitoring describe pod -l app.kubernetes.io/name=loki`, then logs.
2. The usual root cause on a single-node filesystem-backed Loki is its PVC filling: check the PV usage. Retention is short by design; if it filled early, something started logging at volume (find the noisy host/unit on the logs dashboard once ingestion resumes).
3. External Promtails buffer and retry, so a short Loki outage self-heals with a delayed backfill; a long one loses whatever exceeds the agents' buffering.

## PromtailDown

1. Pod state and logs as above. On restart it resumes from its positions file; a wiped positions file re-ships the journal `max_age` window (harmless duplicates, not data loss).
2. If logs are flowing from external hosts but not pods, it's only this DaemonSet: check it mounts the node's log paths and its service account survived whatever changed last.

## Afterwards

Run the fleet drift check ([../verify-promtail-fleet.md](../verify-promtail-fleet.md)): an ingestion outage is a good moment to notice an external agent that never came back.

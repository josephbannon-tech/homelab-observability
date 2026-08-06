# Verify all Promtail agents are shipping

**When**: periodically, and after any host rebuild or Promtail change.

```bash
curl -s http://192.168.0.207:31100/loki/api/v1/label/hostname/values
```

**Expected**: every host in the fleet appears in the label values (seven, in this estate).

Why this works as a drift detector: Loki's retention is 7 days, so a host that stops shipping falls out of this list within a week, with no action needed to maintain the check itself. It costs one command and catches the silent-zero-entries failure mode (see [add-host-logs.md](add-host-logs.md)) at the fleet level.

A missing host means: check `promtail_sent_entries_total` on that host's `:9080/metrics`, then its systemd unit, then the group memberships. The `PromtailDown` alert covers the in-cluster agent; external agents are only caught by this check or by their host's node_exporter-based alerting.

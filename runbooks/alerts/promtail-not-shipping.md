# PromtailNotShipping

| Severity | Condition |
|----------|-----------|
| warning | a host's Promtail has sent no log entries for 30m, sustained 30m |

**Signal**: `PromtailDown` only covers the in-cluster pod. For every other host, the host-state script reads Promtail's own `promtail_sent_entries_total` from `localhost:9080/metrics`, sums it and re-exports it as `homelab_promtail_sent_entries_total` through node_exporter. The rule fires when that counter stops rising, or when the host reports state but no counter at all (Promtail not answering). The script writes one journal line per run, so a healthy host always has something to ship.

**Why re-export and not scrape Promtail directly**: the Promtail port is closed from the cluster on firewalled hosts, while the node_exporter port is already open everywhere. A scrape of `:9080` would also prove only that the process is alive. The counter catches the case that has bitten this estate: Promtail running, healthy by every liveness check, and shipping nothing.

## Triage

1. `systemctl status promtail` on the host. Dead or restarting: read `journalctl -u promtail -n 50`.
2. Running but flat: `curl -s localhost:9080/metrics | grep promtail_sent_entries_total`. No lines at all means it has read nothing since start.
3. Check group membership: `id promtail` must show `systemd-journal` (journal scrape, fails silently without it) and `adm` (text logs under `/var/log`, fails noisily). See [add-host-logs.md](../add-host-logs.md).
4. Check the push side: `promtail_request_duration_seconds_count{status_code!="204"}` rising means Loki is refusing or unreachable; go to [logging-stack-down.md](logging-stack-down.md).
5. Confirm in Loki that the host's stream has resumed before closing.

# Runbooks

One task per file. Each runbook is written to be executed under mild stress: the commands are copy-pasteable, the expected output is stated, and the failure modes that motivated the runbook are described rather than hidden.

## Alert triage (`alerts/`)

One file per alert (or fast/slow pair), linked from every alert's `runbook_url` annotation so the page carries its own triage. Each states what the signal actually measures, how to tell a false positive, and the known causes from this estate's incident history.

| Runbook | Alerts |
|---------|--------|
| [plex-reachability-burn](alerts/plex-reachability-burn.md) | PlexReachabilityBurnFast/Slow (the real user-facing SLI) |
| [plex-library-unreadable](alerts/plex-library-unreadable.md) | PlexLibraryUnreadable (up-but-can't-serve / SMB wedge) |
| [plex-reboot-storm](alerts/plex-reboot-storm.md) | PlexRebootStorm |
| [plex-exporter-burn](alerts/plex-exporter-burn.md) | PlexErrorBudgetBurnFast/Slow (demoted exporter signal) |
| [minecraft-burn](alerts/minecraft-burn.md) | MinecraftErrorBudgetBurnFast/Slow |
| [pihole-dns-burn](alerts/pihole-dns-burn.md) | PiholeDnsBurnFast (the real user-facing SLI) |
| [pihole-exporter-burn](alerts/pihole-exporter-burn.md) | PiholeExporterWedged (exporter signal; replaced the burn-rate pair 2026-09) |
| [nas-burn](alerts/nas-burn.md) | NasErrorBudgetBurnFast/Slow |
| [tailscale-burn](alerts/tailscale-burn.md) | TailscaleNodeBurnFast/Slow |
| [smart-drive-warning](alerts/smart-drive-warning.md) | SmartDriveAttributesBad, SmartSsdLifeLeftLow |
| [smart-drive-failed](alerts/smart-drive-failed.md) | SmartDriveStatusFailed |
| [node-filesystem](alerts/node-filesystem.md) | NodeFilesystemLow/Critical |
| [node-resource-pressure](alerts/node-resource-pressure.md) | NodeMemoryThrottled, NodeIOThrottled |
| [node-clock-skew](alerts/node-clock-skew.md) | NodeClockSkew |
| [zfs-pool-not-online](alerts/zfs-pool-not-online.md) | ZfsPoolNotOnline |
| [backup-heartbeat-stale](alerts/backup-heartbeat-stale.md) | BackupHeartbeatStale, VzdumpHeartbeatStale |
| [jbvm02-maintenance-stale](alerts/jbvm02-maintenance-stale.md) | Jbvm02MaintenanceStale |
| [media-corruption-detected](alerts/media-corruption-detected.md) | MediaCorruptionDetected |
| [jbvm01-nas-dropbox-mount](alerts/jbvm01-nas-dropbox-mount.md) | NasDropboxMountAbsent, NasDropboxMountUnhealthy |
| [tautulli-monitoring-down](alerts/tautulli-monitoring-down.md) | TautulliMonitoringDown |
| [logging-stack-down](alerts/logging-stack-down.md) | LokiDown, PromtailDown |
| [tracing-stack-down](alerts/tracing-stack-down.md) | OtelCollectorDown, TempoDown |

## Change validation (before merge)

- [validate-chart-values-pre-merge.md](validate-chart-values-pre-merge.md): the four-step gate for any chart edit that touches a tool's own config schema
- [validate-blackbox-probe.md](validate-blackbox-probe.md): verifying a new or changed probe produces `probe_success == 1` in live Prometheus before the PR merges

## Verification and testing

- [synthetic-alert-test.md](synthetic-alert-test.md): fire a synthetic alert through Alertmanager to prove the notification path end to end
- [verify-promtail-fleet.md](verify-promtail-fleet.md): one-line drift detector for log shipping across the fleet

## Onboarding a host

- [add-host-metrics.md](add-host-metrics.md): node_exporter + scrape config for a new host
- [add-host-logs.md](add-host-logs.md): Promtail install, including the two group-membership failure modes (one silent, one noisy)

## Recovery procedures

- [force-prometheus-operator-reload.md](force-prometheus-operator-reload.md): when an `additionalScrapeConfigs` change doesn't reach the running Prometheus
- [reload-grafana-dashboards.md](reload-grafana-dashboards.md): manual provisioning reload, including the BusyBox wget gotcha
- [tune-cpu-throttling-alert.md](tune-cpu-throttling-alert.md): diagnosing `CPUThrottlingHigh` on bursty exporters

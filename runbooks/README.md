# Runbooks

One task per file. Each runbook is written to be executed under mild stress: the commands are copy-pasteable, the expected output is stated, and the failure modes that motivated the runbook are described rather than hidden.

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

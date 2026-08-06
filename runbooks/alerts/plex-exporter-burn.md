# PlexErrorBudgetBurnFast / PlexErrorBudgetBurnSlow

| Alert | Severity | Condition |
|-------|----------|-----------|
| PlexErrorBudgetBurnFast | warning | exporter-scrape burn >14.4x on 1h+5m windows, 2m |
| PlexErrorBudgetBurnSlow | warning | exporter-scrape burn >6x on 6h+30m windows, 15m |

**Signal**: `up{job="plex"}`, i.e. **plex-exporter scrape success. This is not the streaming path.** These alerts were deliberately demoted from critical to warning after two false pages: the exporter builds `/metrics` by querying the Plex API for library and session stats, so its scrape latency tracks PMS load, and the scrape times out precisely when Plex is busiest (which is when the family is watching). The user-facing signal is [plex-reachability-burn.md](plex-reachability-burn.md).

## Triage

1. **Verify real Plex first**: `curl -sf http://192.168.0.202:32400/identity`. If that succeeds, no user impact exists; everything below is exporter diagnostics.
2. **Is the reachability alert also firing?** If yes, work that runbook instead; this alert is downstream noise of a real outage.
3. **Exporter pod**: `kubectl -n monitoring get pods -l app=plex-exporter` and logs. The exporter's entrypoint is wrapped in a wait-for-Plex gate, so during a Plex outage it parks `Running`/`NotReady` with zero restarts; a crash-looping exporter with Plex up is a different bug worth looking at.
4. **PMS load**: if scrapes are timing out with Plex healthy, check transcode load and whether a library scan is running. Persistent timeouts under normal load justify raising the scrape timeout, not the alert threshold.

## Do not

- Do not treat this as a Plex outage without step 1.
- Do not relax the burn-rate thresholds to quiet it; the thresholds are correct, and the fix for noise here has always been signal quality or exporter resilience.

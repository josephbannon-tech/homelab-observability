# TautulliMonitoringDown

| Severity | Condition |
|----------|-----------|
| warning | `tautulli_up == 0` or the series absent, sustained 30m |

**Signal**: Plex session/transcode/bandwidth telemetry has stopped flowing. **This is a monitoring-coverage gap, not a Plex outage**: the reachability and library SLIs are independent and unaffected. The 30m `for` is deliberate; Tautulli restarting with Plex briefly away should not page anyone.

## Triage

1. `kubectl -n monitoring get pods` for the tautulli and tautulli-exporter pods; logs for whichever is unhappy.
2. **The known crash-loop trap**: the official Tautulli image is s6-based and **must start as root** (its entrypoint chowns `/config` then drops to uid 1000 itself). A well-intentioned `runAsNonRoot`/`runAsUser` hardening pass crash-loops it with "failed switching to tautulli: operation not permitted". The correct hardening is pod-level `fsGroup` only; the manifest comments say so, believe them.
3. **Exporter up, `tautulli_up=0`**: the exporter can't reach Tautulli's API. Check the API key secret and Tautulli's own connection to Plex (it needs the Plex token; a rotated token breaks it quietly).
4. **After a PVC or pod rebuild**: Tautulli's config lives on its PVC; a fresh PVC means the seeded config must re-establish the PMS connection before metrics resume.

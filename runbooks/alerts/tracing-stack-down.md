# OtelCollectorDown / TempoDown

| Alert | Severity | Condition |
|-------|----------|-----------|
| OtelCollectorDown | warning | `up{job="otel-collector"} == 0`, sustained 5m |
| TempoDown | warning | `up{job="tempo"} == 0`, sustained 5m |

**Signal**: the tracing pipeline's health. The collector is the only externally-exposed trace entrypoint (OTLP NodePorts 30317/30318); Tempo stores and serves what the collector forwards. These alerts matter because an off-cluster producer (the NAS media pipeline) depends on the collector.

**No workload is at risk.** The bash-pipeline producer is strictly fail-open by design: with the collector down it behaves identically to tracing-disabled (proven by its test harness). What is lost is the spans themselves; there is no buffering, so traces from the outage window are gone.

## Triage

1. `kubectl -n monitoring describe pod` for the collector (`app.kubernetes.io/name=opentelemetry-collector`) or Tempo, then logs.
2. **Collector up but producers failing to export**: check the NodePort service still exists and the port matches what producers use (NodePorts must sit in 30000-32767; a config typo here once produced an ArgoCD SyncError rather than a runtime failure, so also glance at app sync state).
3. **TempoDown with the collector up**: the collector logs export failures to Tempo. Check Tempo's PVC (filesystem-backed, same failure shape as Loki: it fills, it stops).
4. **Verify end to end after recovery**: send a one-shot test span (grpcurl OTLP export against the collector service) and confirm it lands in a Tempo search, rather than trusting `up` alone.

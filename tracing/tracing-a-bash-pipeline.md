# Tracing a bash pipeline without an SDK

The estate's media-integrity pipeline is a bash program on the NAS: it verifies downloaded media by decoding it in a k8s Job, hardlinks clean files into the library, and deletes confirmed-corrupt ones. It is also the estate's first distributed-trace producer. Each processed file emits one trace, correlatable to its logs and metrics in Grafana, and the instrumentation adds no runtime dependency beyond a single static binary.

## Why bother tracing a shell script

The pipeline spans two machines and three execution contexts: an orchestrator on the NAS, an ffmpeg scan Job on the k3s cluster, and follow-up actions back on the NAS. When a file takes 40 minutes to process, "which stage, on which host, and what did it decide" is exactly the question traces answer and logs make you reconstruct by hand. It was also a deliberate exercise: the Tempo + OTel Collector half of the stack was deployed and idle, and an instrumented real workload proves the pipeline end to end in a way a test span does not.

## Mechanics

- **Tool**: [`otel-cli`](https://github.com/equinix-labs/otel-cli), a single static Go binary. No Python, no SDK, nothing to pip-install on an appliance OS with a read-only root filesystem.
- **Transport**: OTLP/HTTP to the OTel Collector's NodePort, forwarded to Tempo. The collector is the only externally-exposed trace entrypoint.
- **Span model**: one root `ingest-item` span per scanned file (attributes: relative path, category, verdict) enclosing `scan` (with exit code), `hardlink`, and `handle-corrupt` children. Span attributes carry the pipeline's actual decisions, so a trace reads as a verdict record.

## Trace-to-logs across two hosts

The pipeline's `log()` function prefixes active lines with `[trace=<id>]`, so the NAS journal lines for a file are greppable by trace id once Promtail ships them to Loki. The W3C `traceparent` is propagated into the k8s scan Job via its environment, and the pod echoes the trace id into its own logs. Grafana's `tracesToLogsV2` correlation uses filter-by-trace-id matching on line content, which needs no Loki label extraction: clicking a span jumps to the matching log lines on either host.

This is the poor man's context propagation, and that's the point: W3C trace context is just a string. Any process that can pass an environment variable and echo it can participate in a distributed trace.

## Strictly fail-open, and proven to be

The pipeline's corrupt-path deletes data, so instrumentation must never influence control flow. Every trace call swallows errors (`|| true`, a 1-second export timeout) and never contributes to a verdict. There are three independent off-switches: an environment variable, the binary being absent, and a flag file beside the binary.

The property is tested, not asserted: an 18-case test harness includes running the full pipeline with the collector down and asserting behaviour is byte-identical to tracing-disabled. "Fail-open" claims that aren't exercised in CI are wishes. The collector and Tempo have their own `up`-based health alerts, so a dead trace pipeline is a monitoring page, never a workload incident.

## What this demonstrates

- OTLP instrumentation is not tied to application frameworks; infrastructure glue code can emit first-class traces.
- Trace context propagation through env vars + log echoing gives cross-host correlation with zero library support.
- Observability code in a destructive path must be provably inert, and the proof belongs in the test suite.

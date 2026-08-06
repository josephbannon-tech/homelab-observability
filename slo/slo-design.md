# SLO design for family services

The estate's user-facing services carry stated availability targets, enforced as Prometheus recording rules and multi-window multi-burn-rate alerts (Google SRE Workbook, chapter 5). This document covers how the SLIs were chosen, and specifically the three failures that shaped the current design. The live rules are in [homelab-gitops `charts/kube-prometheus-stack/values.yaml`](https://github.com/josephbannon-tech/homelab-gitops/blob/main/charts/kube-prometheus-stack/values.yaml) under `additionalPrometheusRulesMap`.

## Targets

| Service | SLI | Target |
|---------|-----|--------|
| Plex (reachability) | blackbox HTTP probe of `/identity` | 99.5% (service-hours) |
| Plex (library readable) | authed `GET /library/sections` custom probe | alert-only |
| Minecraft | `minecraft_status_healthy` (server-list-ping) | 99.0% |
| Pi-hole DNS | blackbox DNS probe, real lookup on UDP/53 | 99.9% |
| NAS reachable | node_exporter scrape success | 99.9% |
| Tailscale nodes | tailscaled debug-metrics scrape, per node | 99.0% |

Burn-rate alerting per service: fast burn at 14.4x over 1h+5m windows pages at critical; slow burn at 6x over 6h+30m windows warns. The thresholds are the SRE Workbook values and are deliberately never relaxed to suppress noise: in every noisy incident so far the problem was the signal, not the sensitivity.

## Lesson 1: an exporter's `up` is not a user-facing SLI

Both Pi-hole and Plex originally shipped with `up{job="<exporter>"}` as their SLI, and both false-paged at critical severity while the actual service was healthy.

The mechanism is the same for any exporter that builds `/metrics` by querying its target's API: the scrape's latency tracks target load, so the scrape hits the 10s timeout precisely when the target is busy, and `up` flips to 0 while users are fine. Pi-hole's exporter also sits on a different path from the DNS listener entirely, so the two can fail independently in either direction.

The convention that came out of it, applied to every service since:

1. **Pick an SLI that exercises the real user path**: a blackbox probe of the endpoint users actually hit. Choose an endpoint that does no heavy work (Pi-hole: a real DNS lookup on UDP/53; Plex: `GET /identity`, which touches no library or session data) so it stays green when the service is busy but reachable.
2. **Give the real SLI its own rule group and metric prefix** (`slo:pihole_dns:*` vs `slo:pihole:*`) so dashboards and alerts pick either signal explicitly, never by accident.
3. **Keep the exporter alert but demote it** to warning severity and reword it as "exporter scrape failing", an exporter-path diagnostic rather than a user-impact page.
4. **Repoint the user-facing dashboard tiles** at the new recording rules and remove the lying tile.

The layering matters as much as the probe choice. Plex now has three signals with distinct meanings: L2 reachability (`/identity` up), L3 serviceability (an authenticated `GET /library/sections`, which exercises the PMS database and the backing media store), and session telemetry via Tautulli. The L3 probe exists because of a real outage the L2 probe couldn't see: the Plex host's SMB mount to the NAS came back from a reboot wedged, "connected" but reading 0 bytes, so Plex answered `/identity` cheerfully while nothing would play. The household found it before the monitoring did. A probe that only proves the process is alive can't catch up-but-can't-serve.

## Lesson 2: an honest SLO needs a maintenance window, and it can live in PromQL

The Plex host (an Android TV device) performs a nightly self-reboot around 2am that survives every attempt to disable it. Raw 30-day availability sat at ~96.5%, and 40 of 64 recorded "outage episodes" over a week were single 30-second blips from that reboot. The SLO was numerically true and operationally useless: it hid real regressions inside routine noise and paged people for a reboot nobody could prevent.

Three rules fixed it, all in PromQL rather than Alertmanager (deliberately: the Alertmanager config lives in an out-of-git Secret, so logic placed there is invisible to review; recording rules are in version control):

- `plex:hard_down`: no successful probe for a full 3 minutes. Hysteresis that ignores single-sample blips.
- `plex:maintenance_window`: a time-of-day expression marking the nightly reboot window. It gates the reachability burn alerts so routine reboots don't page.
- `slo:plex_http:availability:ratio_30d_svchours`: the headline SLO, excluding the maintenance window. Result: ~99.8% service-hours availability against ~96.5% raw, and the number now moves when something real breaks.

The trade-off is stated openly: a rare hang during the window surfaces at window-end rather than 02:15, accepted because household impact at that hour is nil. Two alerts deliberately ignore the gate: the L3 library probe (storage breakage is never routine) and a reboot-storm detector (more than 4 reachability drops in 3h, which catches a failed-update retry loop that once produced 8 drops in two hours).

## Lesson 3: aggregate away churny labels before recording an SLO

Found while extracting this repo, fixed in [homelab-gitops PR #59](https://github.com/josephbannon-tech/homelab-gitops/pull/59). The Minecraft monitor labels its health metric with `server_version`, discovered by pinging the server. When the server is down there is nothing to ping, so the metric is emitted without that label. Two label sets means two series: one that only exists while healthy (value 1) and one that only exists while down (value 0).

`avg_over_time(minecraft_status_healthy[30d])` recorded both identities verbatim. The result on the SLO board was a real 100% tile next to a phantom 0% tile, the second series' average being 0 by construction and pinned there for a full 30-day window after any outage.

The fix merges the identity before averaging:

```promql
avg_over_time((max without (server_version) (minecraft_status_healthy))[30d:5m])
```

`max` is exact here, not an approximation, because Prometheus writes staleness markers the moment a series vanishes from a scrape, so the two shapes hand over cleanly and never overlap. The general rule: any label whose presence depends on the target's state must be aggregated away before the value enters a recording rule, or the rule forks into per-state series.

## Verification discipline

Every one of these rules is validated against live Prometheus before merge, because two probes shipped silently broken behind green ArgoCD/pod statuses (an NXDOMAIN-answering test query, and a misspelled config field that crash-looped only the new ReplicaSet). The procedure is [runbooks/validate-blackbox-probe.md](../runbooks/validate-blackbox-probe.md); the broader chart-edit gate is [runbooks/validate-chart-values-pre-merge.md](../runbooks/validate-chart-values-pre-merge.md).

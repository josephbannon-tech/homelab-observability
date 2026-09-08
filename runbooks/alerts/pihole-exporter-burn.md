# PiholeExporterWedged

| Alert | Severity | Condition |
|-------|----------|-----------|
| PiholeExporterWedged | warning | `up{job="pihole"} == 0` or `scrape_duration_seconds{job="pihole"} >= 9.9`, sustained 15 min |

**Signal**: pihole-exporter scrape health, **not user-facing DNS.** The exporter polls Pi-hole's admin API on port 80; the DNS listener on `:53` is an independent path, and the household signal is [pihole-dns-burn.md](pihole-dns-burn.md). This alert replaced a pair of error-budget burn-rate alerts on the same signal (`PiholeErrorBudgetBurnFast/Slow`). Those were retired because a 99.9 % objective on an exporter fires on a single self-healing restart, and because the exporter's real failure mode is a bug that the liveness probe now recovers on its own. This alert means the probe has evidently *not* recovered it, so a human has to.

## Triage

1. **Verify real DNS first**: `dig @192.168.0.205 +short google.com`. Success means zero user impact; the rest is exporter diagnostics.
2. **Confirm the wedge fingerprint**: `scrape_duration_seconds{job="pihole"}` jumped from ~0.25 s to a flat 10.0 s (the scrape timeout ceiling) at a clean cliff, the pod is `1/1 Running` with no recent restarts, and Pi-hole's API answers instantly when you hit it directly (`curl -s -o /dev/null -w '%{http_code} %{time_total}\n' http://192.168.0.205/api/stats/summary` returns 401 in under a millisecond). If instead the API itself is slow or erroring, this is a Pi-hole problem: check `pihole-FTL` on the LXC and stop here.
3. **Recover**: `kubectl -n monitoring delete pod -l app=pihole-exporter`. Do not `rollout restart` (it patches the template and the GitOps controller sees drift). `up` returns within one scrape.
4. **Then find out why liveness did not fire.** The deployment's only probe is an httpGet liveness on `/metrics` with a 5 s timeout and `failureThreshold: 2`. `kubectl describe pod` should show probe failures leading to a restart within about a minute of the wedge. If it does not, someone has changed the probe (raised the threshold, lengthened the timeout, or re-added a readiness probe) and the design note below explains why each of those breaks recovery.

## Why the exporter wedges (v1.2.0, upstream issue eko/pihole-exporter#332)

The `/metrics` handler collects under a 10 s context. On timeout it spawns a goroutine to discard the late result from the client's `Status` channel, but then still blocks reading that same channel itself. The channel is one per Pi-hole client, capacity 1, shared by every request. After a single collection exceeds 10 s, the discard goroutine and the handler race for every subsequent result, so each `/metrics` call returns only when the *next* caller's collection completes. Traced on 2026-09-07 with strace and tcpdump inside the pod's network namespace: all seven Pi-hole API calls finished in 0.18 s, and the response was written 16 s later at the exact moment the kubelet probe's collection landed. The chain is self-sustaining for as long as anything keeps calling `/metrics`. Fix PRs exist upstream (#328, #330) and a fixed fork; the maintainer has been inactive since the v1.2.0 release.

## Probe design note (read before changing the probes)

- **A readiness probe makes it worse.** It is a third caller feeding the chain and gates nothing, because Prometheus scrapes the pod IP via Endpoints and no traffic routes through the Service. Removed.
- **A high liveness `failureThreshold` can never trip.** With several callers on the shared channel, whichever waiter is present when a result lands wins, so some probe always gets a fast answer and six consecutive failures never happen (observed: 27 hours wedged, zero restarts, under `failureThreshold: 6`). With only two callers (Prometheus and liveness) a wedged pod answers the probe about 30 s late, the 5 s timeout fails twice in a row, and kubelet restarts it in about 60 s.
- **The exporter's built-in `/liveness` and `/readiness` routes always return 200.** They are useless for this.
- **Cost of the tight probe**: a genuine Pi-hole API stall longer than 5 s on two consecutive probes also restarts the exporter. That is harmless (exporter-only blast radius) and, at a 99.9 % objective, was the reason the burn-rate alerts had to go: two lost scrapes in an hour is a 16.7x burn.

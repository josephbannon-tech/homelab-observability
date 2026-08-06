# PiholeErrorBudgetBurnFast / PiholeErrorBudgetBurnSlow

| Alert | Severity | Condition |
|-------|----------|-----------|
| PiholeErrorBudgetBurnFast | warning | exporter-scrape burn >14.4x on 1h+5m windows, 2m |
| PiholeErrorBudgetBurnSlow | warning | exporter-scrape burn >6x on 6h+30m windows, 15m |

**Signal**: `up{job="pihole"}`, i.e. **pihole-exporter scrape success, not user-facing DNS.** The exporter polls Pi-hole's admin API; the DNS listener on `:53` is an entirely independent path. These alerts false-paged at critical on day one (the exporter's own liveness probe tripped on a slow admin API and restarted a healthy pod) and were demoted to warning. The household signal is [pihole-dns-burn.md](pihole-dns-burn.md).

## Triage

1. **Verify real DNS first**: `dig @192.168.0.205 +short google.com`. Success means zero user impact; the rest is exporter diagnostics.
2. **Known wedge mode**: this exporter version can wedge against the Pi-hole v6 API: `up=0` or `pihole_up=0`, `scrape_duration_seconds` pinned at exactly the 10s timeout, DNS completely fine, and HTTP probes on the pod all green (so kubelet never restarts it). **Fix: delete the pod.** It recreates and reconnects.
3. **Admin API health**: if the exporter is healthy but scrapes still fail, check Pi-hole's web/API service on the LXC (port 80) separately from FTL.
4. **Probe design note**: the pod's liveness probe is a TCP socket check, deliberately not HTTP, because an HTTP liveness probe on a proxy-style exporter turns upstream slowness into kubelet restart loops. If someone "fixes" it back to HTTP, expect the day-one false page to return.

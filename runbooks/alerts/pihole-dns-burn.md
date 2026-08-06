# PiholeDnsBurnFast

| Severity | Condition |
|----------|-----------|
| critical | blackbox DNS probe burn >14.4x on 1h+5m windows, sustained 2m |

**Signal**: the blackbox `dns_pihole` probe performing a real `google.com` lookup against `192.168.0.205:53/udp`. This is the household-impacting signal: if it burns, every device on the LAN is failing to resolve names. (The probe queries a real name deliberately; an earlier version queried a synthetic name with no record, got NXDOMAIN forever, and was dead for nine days without anyone noticing.)

## Triage

1. **Confirm from another host**: `dig @192.168.0.205 +short google.com`.
2. **Scope it: one client or everyone?** If a single machine's DNS is broken while this probe is green (or the reverse), check per-client rate limiting first: Pi-hole's default 1000 queries/min per client answers REFUSED to **all** of that client's queries once tripped. `grep -i rate FTL.log`. A bulk download or a machine with no local DNS cache trips it.
3. **Service**: `systemctl status pihole-FTL` on JBDNS01. Remember it's an unprivileged LXC: its clock is host-managed and in-app NTP must stay off; a container restart from the host side is the recovery lever if SSH is unresponsive.
4. **Local resolution vs upstream forwarding**: if local names resolve but internet names fail, the fault is upstream. Check the Pi-hole admin UI's upstream servers and FTL's upstream error counters. **High blocked-query volume is never the cause**: blocked queries are answered locally and don't touch the WAN; look at `queries_forwarded` and upstream failures instead.
5. **Upstream/WAN**: TCP forward failures to both upstreams simultaneously is the WAN-outage fingerprint, not a Pi-hole fault. Check the router/modem's event log (DOCSIS T3 timeouts and re-registrations in this estate's case) before touching DNS config.

## Bear in mind

DNS is the dependency of everything else here; during a real DNS outage other alerts fan out. Fix DNS first and let the rest resolve, and be aware the LAN keeps limping on client caches for a few minutes, so user reports lag the probe.

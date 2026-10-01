# WanDown / WanPartialService / UpstreamDnsDown

| Alert | Severity | Condition |
|-------|----------|-----------|
| WanDown | warning | ICMP to both anycast resolvers (1.1.1.1, 9.9.9.9) failing for 5m |
| WanPartialService | warning | HTTPS to a connectivity-check endpoint failing for 10m while ICMP still passes |
| UpstreamDnsDown | warning | DNS to both upstream resolvers failing for 5m while ICMP passes |

**Signal**: three blackbox probes from the cluster through the household hub. They ran info-only for 21 days before becoming alerts (99.96 to 99.997% per target, never both ICMP targets down for a full two minutes), so `for: 5m` on both targets had zero false positives in the baseline. The Pi-hole DNS SLI alone cannot see a WAN outage: it answers from cache until TTLs expire.

**Delivery caveat**: while the WAN is down Alertmanager cannot reach the chat webhook, so `WanDown` arrives when the line returns. It is still the timestamped record for the ISP, and an inhibition rule mutes the three sibling alerts (`WanPartialService`, `UpstreamDnsDown`, the Pi-hole DNS burn pair) so the return produces one message, not four.

## Triage

1. **Which shape?** `WanDown` = the line is gone. `WanPartialService` = pings pass but TCP/TLS stalls, the DOCSIS partial-service fingerprint (missing upstream channels). `UpstreamDnsDown` = UDP/53 outbound is being dropped while the line otherwise works.
2. **Hub event log and channel table** at the hub's LAN address (unauthenticated on the Virgin Media hub; timestamps are UTC). Note the first event time for the ISP record.
3. **Confirm from a LAN host, not the cluster**: `ping -c3 1.1.1.1`, `curl -sI https://connectivitycheck.gstatic.com/generate_204`, `dig @1.1.1.1 google.com`. If the LAN host succeeds and the cluster does not, the fault is on the cluster node's path, not the WAN.
4. **UpstreamDnsDown only**: a hub reboot has cleared UDP/53 drops before. Pi-hole itself is fine if `blackbox-dns-pihole` still reads 1.
5. **Persistent partial service**: this is the evidence for a reprovision request to the ISP; keep the probe history (Grafana) and the hub log.

## Do not

- Do not reboot Pi-hole for a WAN alert; it is the messenger. The cache is what keeps the LAN usable for a while.

# WanBufferbloat

| Alert | Severity | Condition |
|-------|----------|-----------|
| WanBufferbloat | warning | p95 ICMP RTT to both anycast resolvers (10 min window) above 39 ms for 15 min, with no failed probe in the window |

**Signal**: the same `wan-icmp` blackbox probes that feed `WanDown`, read for latency instead of success. A saturated uplink queues packets in the cable modem; round trips triple before anything fails. Reachability probes stay green the whole time, which is how a week of game and Discord drops went unalerted in October 2026. The idle floor is 13 to 18 ms; the incident week sat at 38 ms with 45% of ten-minute buckets over the threshold. Three weeks after the uploader was capped: zero firings.

**Why the guard**: one 5 s probe timeout inside the window drags the p95 to 5 s and says nothing about queueing. Windows holding a failed probe are excluded; that shape belongs to `WanDown` and `WanPartialService`. `keep_firing_for: 30m` stops a loaded evening from producing a message every twenty minutes.

## Triage

1. **Is the estate the uploader?** Hypervisor WAN-facing NIC transmit rate, in Mbit/s: `rate(node_network_transmit_bytes_total{hostname="<hypervisor>",device="nic0"}[1m])*8/1e6`. Sustained above half the uplink (about 19 Mbit/s on a 38.5 Mbit/s line) means a bulk uploader on the LAN side.
2. **Find it**: the torrent client's global upload cap (`/api/v2/app/preferences | jq .up_limit`, expected 20 Mbit/s or lower); off-site backup jobs; phone photo sync and file sync, both of which upload in bulk after a reconnect. Per-guest transmit rates narrow it to a VM.
3. **Cap it, do not kill it**: every bulk uploader on the estate carries a rate cap below half the uplink. A missing cap is the defect; add one and record it in the host doc.
4. **NIC idle, latency high**: the queue is upstream. Take a DOCSIS snapshot from the hub (channel table, event log, `US profile assignment change` events) and correlate with the alert window. Persistent, load-independent latency is ISP evidence.
5. **Confirm the user-facing effect** before closing: a game or voice session from a LAN host during the alert, not a speed test. Speed tests measure throughput, which bufferbloat leaves intact.

## Do not

- Do not raise the threshold to quieten a loaded evening. The alert is the capacity signal; the fix is a cap on the uploader.
- Do not treat a passing `WanDown` as proof the line is fine. That alert measures reachability; this one measures the queue in front of it.

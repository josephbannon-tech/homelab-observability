# TailscaleNodeBurnFast / TailscaleNodeBurnSlow

| Alert | Severity | Condition |
|-------|----------|-----------|
| TailscaleNodeBurnFast | critical | burn >14.4x on 1h+5m windows, 2m |
| TailscaleNodeBurnSlow | warning | burn >6x on 6h+30m windows, 15m |

**Signal**: scrape success of `tailscaled --debug` metrics (`:5252`) on the nodes that run Tailscale (JBVM03, JBDNS01). JBDNS01 is also the subnet router, so its node burning means remote access to the whole LAN is impaired, not just one host.

## Triage

1. **Service**: `systemctl status tailscaled` on the named host, then its journal. On the LXC, remember `/dev/net/tun` passthrough is required; a container rebuild that loses it produces tailscaled failing at start.
2. **Distinguish daemon-down from metrics-down**: `tailscale status` working while `:5252` refuses means only the debug listener died (restart the unit); connectivity is fine.
3. **DERP re-home / flapping** (slow burn): the journal shows repeated DERP relay changes. Two environmental causes dominate here:
   - **WAN saturation**: a multi-GB download saturating the downlink causes bufferbloat that drops Tailscale's UDP paths; exit-node clients lose all internet while the LAN looks healthy. Check WAN throughput at the time of the flaps; the fix is throttling bulk pulls, not touching Tailscale.
   - **Upstream/ISP instability**: correlate flap timestamps with the modem's event log before blaming the node.
4. **Forwarding performance complaints** (not this alert, but reported alongside): any node forwarding traffic needs UDP GRO forwarding configured on its NIC (`ethtool -K <if> rx-udp-gro-forwarding on rx-gro-list off`), persisted via a systemd unit so it survives reboots.

## Client-side gotchas that are NOT node faults

- A device that is physically on a subnet the tailnet advertises must not accept routes for its own LAN, or it tunnels its local traffic and breaks LAN-local connections.
- Shared nodes get a different 100.x address in each recipient tailnet; a friend's "can't connect" with your side green is an addressing question, not a node outage.

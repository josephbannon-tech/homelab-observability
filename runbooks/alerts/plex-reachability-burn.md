# PlexReachabilityBurnFast / PlexReachabilityBurnSlow

| Alert | Severity | Condition |
|-------|----------|-----------|
| PlexReachabilityBurnFast | critical | burn rate >14.4x on 1h+5m windows, sustained 2m |
| PlexReachabilityBurnSlow | warning | burn rate >6x on 6h+30m windows, sustained 15m |

**Signal**: the blackbox HTTP probe of `http://192.168.0.202:32400/identity`. This is the real user-facing SLI: family devices cannot reach Plex. Both alerts are suppressed during the 02:00-06:00 BST maintenance window (the host's nightly self-reboot), so if this is paging, it is outside the routine reboot and real.

## Triage

1. **Confirm from another host**: `curl -sf http://192.168.0.202:32400/identity`. Success returns XML with a `machineIdentifier`.
2. **Is the Shield host alive at all?** Check ICMP, and check whether the device is still making outbound DNS queries in the Pi-hole query log. A host that queries DNS but times out on `:32400` AND ICMP is the **stuck mid-boot signature**, not a clean power-off.
3. **Power/network**: if genuinely dark, check power and the switch port before anything software.

## Known causes, in likelihood order

- **Nightly reboot hang.** The Shield self-reboots around 2am nightly (persists even with OTA endpoints blocked at the DNS layer). Normally back in under a minute; occasionally it hangs mid-boot and needs a manual power-cycle. An alert shortly after the maintenance window ends is very likely this, surfacing late because paging was suppressed inside the window.
- **Network drop on the device.** A ~33-minute outage once fired this alert with the host simply off the network; the DNS probe on the same blackbox exporter stayed green, proving the monitoring path was fine.
- **PMS crash with host up**: ICMP fine, `:32400` refused. Restart Plex on the device.

## After ANY outage

Run a forced full Plex library scan. The media pipeline keeps ingesting during Plex outages, but its per-file scan trigger is fire-and-forget and does not retry, so anything added while Plex was down sits on disk unindexed until a manual scan (Plex UI → library → Scan Library Files, or the authenticated `/library/sections/<id>/refresh?force=1` endpoint).

Also check for the up-but-broken sibling failure: if playback fails after the probe goes green, see [plex-library-unreadable.md](plex-library-unreadable.md), the SMB-wedge case this probe cannot see.

# NodeClockSkew

| Severity | Condition |
|----------|---------|
| warning | `node_timex_sync_status == 0` for the whole of a 5m window, sustained 10m |

**Signal**: the kernel reports the clock is not NTP-synchronised. Skewed clocks corrupt log correlation (the thing you need most mid-incident), TLS validation, and every age-based tile (backup heartbeats read as stale or impossibly fresh). This is a curated replacement for the upstream clock alert lost when the built-in node-exporter alert group was culled.

## Triage

1. `timedatectl status` on the instance, then the time-sync service (`chrony` or `systemd-timesyncd`): is it running, and can it reach its sources?
2. **DNS dependency**: NTP pool hostnames need DNS; a host that lost DNS quietly loses time sync later. Check resolution before restarting the sync daemon.
3. **Firewall**: UDP/123 outbound blocked shows as reachable-but-never-synced sources.

## Platform quirks that produce false trails

- **Unprivileged LXC containers**: the clock is host-managed; in-app or in-container NTP daemons must be disabled (they cannot set the clock and some log errors forever). If an LXC skews, fix the **host's** sync, not the container's.
- **Appliance OSes can lie in `timedatectl`**: on TrueNAS, `timedatectl` has cached stale timezone/sync state; trust `date` and the platform's own API/middleware over it.
- **VM time after migration/suspend**: a guest that resumed with a big offset may need a step (`chronyc makestep`) rather than waiting for slew, which caps at tiny corrections per second and can take days to close a large gap.

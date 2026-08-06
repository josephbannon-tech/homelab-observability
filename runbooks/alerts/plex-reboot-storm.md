# PlexRebootStorm

| Severity | Condition |
|----------|-----------|
| warning | more than 4 reachability-probe drops within 3h, sustained 5m |

**Signal**: `resets(probe_success{job="blackbox-plex"}[3h]) > 4`. One nightly reboot produces one drop; more than four in three hours means the host is cycling abnormally. Deliberately **not** gated by the maintenance window: a storm *inside* the window is exactly what it exists to catch (the reference case was 8 drops between 03:00 and 05:00 from a failed OTA update retry-looping).

## Triage

1. **Count and time the drops**: graph `probe_success{job="blackbox-plex"}` over the last 6h. Regular ~30min spacing points at an update retry loop; irregular clustering points at power/network flapping.
2. **On the device**: check system update status and reboot history. A stuck OTA needs intervention on the device itself (let it finish on good network, or clear the update cache).
3. **Rule out environment**: switch port, PoE budget if applicable, power strip. A flapping network link produces the same probe signature as reboots.
4. **Correlate with DNS logs**: the device's boot fingerprint is visible as a burst of connectivity-check queries in the Pi-hole query log; matching bursts to probe drops distinguishes true reboots from network drops (host up, queries continuous, probe flapping = network, not reboots).

## After it settles

Forced full Plex library scan if any media landed during the storm, and check [plex-library-unreadable.md](plex-library-unreadable.md)'s SMB-wedge fingerprint, since repeated reboots multiply the chance of a wedged mount.

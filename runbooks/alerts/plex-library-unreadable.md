# PlexLibraryUnreadable

| Severity | Condition |
|----------|-----------|
| critical | L3 library probe failing for 5m **while** the L2 `/identity` probe is green |

**Signal**: the custom library probe (an authenticated `GET /library/sections`, which exercises the PMS database and the backing media store) is failing while plain reachability is fine. This is the **up-but-can't-serve** detector: Plex answers, nothing plays. Deliberately not suppressed during the maintenance window, because storage breakage is never routine.

## Why this alert exists

In a 2026-06-07 incident, the Plex host's SMB mount to the NAS came back from its nightly reboot **wedged**: TCP session established, mount "connected", but every read returned 0 bytes. Plex browsed its cached library happily, `/identity` stayed green, and the household discovered the outage by pressing play. This alert closes that gap.

## Triage

1. **Confirm**: `curl -s -H 'X-Plex-Token: <token>' http://192.168.0.202:32400/library/sections` (the token is in the sealed `plex-exporter-config` Secret).
2. **Check the wedge on the NAS side**: on the NAS, `smbstatus` and look at the session for the Shield's IP. A wedged client shows an ESTABLISHED session that serves no reads. Also verify the session authenticates as the intended unprivileged user, not root.
3. **Fix is on the CLIENT**: reboot the Shield (or remount its SMB mount). The NAS is almost certainly fine; do not restart NAS services first.
4. **Rule out the NAS anyway**: `zpool status` and an SMB read from any other client. If other clients also fail, this is a NAS incident, not a wedge.

## After recovery

Forced full Plex library scan (see [plex-reachability-burn.md](plex-reachability-burn.md), same reason: fire-and-forget scan triggers don't retry).

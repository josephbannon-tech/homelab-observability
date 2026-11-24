# SatisfactoryDown

| Severity | Condition |
|----------|-----------|
| warning | health check failing for 15m (probe every 60s) |

**Signal**: a blackbox probe POSTs the `HealthCheck` function to the dedicated server's HTTPS API and requires `"health":"healthy"` or `"slow"` in the answer. The game itself runs over UDP, which a probe cannot judge; the API is served by the same process on the same port number, so a healthy answer means the server is up and ticking, not that a port happens to be open. The certificate is self-signed, so the probe skips verification and the body match is the assertion.

Before this probe the only signal for the host was its save-backup heartbeat. That proves the save files exist. It says nothing about whether anyone can join.

## Triage

1. On the game host: `systemctl status satisfactory` and `journalctl -u satisfactory -n 50`.
2. Check the listeners: the server needs its game port on UDP and TCP **and** the reliable-messaging port on TCP. With only the first one open, clients join and drop with "connection lost" about 24 seconds later, and this probe still passes. If players report that pattern while the alert is quiet, check the second port and any ACL in front of it.
3. A game update that changes the API can fail the body match while the server works. `curl -sk -X POST -H 'Content-Type: application/json' -d '{"function":"HealthCheck","data":{"clientCustomData":""}}' https://<host>:7777/api/v1` shows the raw answer.
4. The save is not at risk from an outage: the nightly archive is taken from disk and does not need the server process. If the world itself is damaged, restore from the newest archive per the backup runbooks.

## Not covered

The probe runs from inside the LAN. Players reach the server over the tailnet, so a broken share or ACL on that path is invisible here.

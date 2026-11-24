# SystemdUnitFailed

| Severity | Condition |
|----------|-----------|
| warning | a systemd unit in the `failed` state, sustained 15m |

**Signal**: the host-state script lists failed units every 5 minutes and writes one `homelab_systemd_unit_failed{unit="..."}` series per unit, plus `homelab_systemd_failed_units` as a count that is always present. A unit that recovers or is reset disappears from the file on the next run, so the alert resolves without help.

This alert exists because `logrotate.service` failed every night for three months on the hypervisor and nothing noticed: the exporter was up, the host was up, every dashboard was green.

**Why a script and not node_exporter's systemd collector**: the collector emits five series per unit, several thousand per host, to answer a question that needs one series per failed unit. It also needs D-Bus, which an exporter running in a pod does not have. The script behaves the same on VMs, containers, the storage appliance and the Kubernetes node.

## Triage

1. `systemctl status <unit>` and `journalctl -u <unit> -n 50` on the host named in the alert.
2. A timer-driven unit will fail again on its next run, so fix the cause before clearing the state.
3. `systemctl reset-failed <unit>` once fixed. The series goes within 5 minutes.

## Known causes here

- A backup copy left inside a config directory that the service parses (`/etc/logrotate.d/foo.bak` is read as a second config for the same log). Keep backups of config files outside the directory they came from.
- A cosmetic unit that fails on every boot of an appliance OS. Put it in the script's `IGNORE_UNITS` for that host, with a comment saying why, instead of silencing the alert.

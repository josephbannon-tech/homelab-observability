# HostRebootPending

| Severity | Condition |
|----------|-----------|
| warning | reboot needed for more than 7 days, sustained 1h |

**Signal**: a host-state script on every host writes `homelab_reboot_required` (1 when the distro's `/var/run/reboot-required` flag exists, or when the newest kernel in `/boot` is not the one running) and `homelab_reboot_required_since_seconds` (the mtime of that flag or kernel). The rule compares the timestamp with `time()` instead of using a 7-day `for:`, so a Prometheus restart does not reset the clock. Containers report the flag only: they run the host's kernel.

This alert exists because unattended upgrades install fixes that a running kernel never picks up. A 2026-11 documentation audit found five hosts in that state at once; no alert had said so.

## Triage

1. `cat /var/run/reboot-required.pkgs` on the host names the packages that asked for it. No flag file means the kernel comparison tripped: `ls /boot/vmlinuz-*` against `uname -r`.
2. Decide the window. A family-facing service host waits for a quiet hour; the hypervisor takes every guest with it.
3. Run the reboot pre-flight before any production reboot: autostart and boot order, DNS dependency, mounts, enabled services, SSH by IP.
4. After the reboot the metric drops to 0 on the next 5-minute run and the alert resolves by itself.

## False positives

- A kernel package installed and then held or pinned on purpose keeps the comparison true. Remove the unused kernel, or accept the alert as the reminder it is.
- An appliance OS that manages its own updates should not report this metric at all; the script takes `REBOOT_CHECK=0` for that host.

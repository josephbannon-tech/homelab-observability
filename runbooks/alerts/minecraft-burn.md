# MinecraftErrorBudgetBurnFast / MinecraftErrorBudgetBurnSlow

| Alert | Severity | Condition |
|-------|----------|-----------|
| MinecraftErrorBudgetBurnFast | critical | burn >14.4x on 1h+5m windows, 2m |
| MinecraftErrorBudgetBurnSlow | warning | burn >6x on 6h+30m windows, 15m |

**Signal**: `minecraft_status_healthy` from mc-monitor on JBVM03 (`:9150`), which pings the server-list-ping endpoint (no RCON). Fast burn means the server is not answering pings and active players are being disconnected or cannot join. Slow burn usually means a restart loop or flapping.

## Triage

1. **Service state**: `systemctl status minecraft` on JBVM03 (192.168.0.204), then the server log and the systemd journal for crash/restart evidence.
2. **Host pressure first**: `uptime` and `free -h` on JBVM03. A modded server can take the whole host down with it; load far above core count with the JVM at max heap is a host-overload incident, not a clean service crash.
3. **JVM state**: if the process is up but not answering, it's usually GC death-spiral or a deadlocked tick. A thread dump (`jstack` or `kill -3`) before restarting preserves the evidence; restarting first destroys it.
4. **Restart loop** (slow burn): check whether the launcher is respawning a crashing server. The crash report in the server's `crash-reports/` names the offending mod more often than the log tail does.

## Known causes from this estate's history

- **Conflicting chunk/light optimisation mods.** The worst incident (load 28 on a 4-core host, all players rubber-banding, OOM risk) root-caused via thread dump to two chunk engines active at once, one of them embedded inside another mod. Never run two optimisation mods that claim the same subsystem; check embedded libraries before adding a new one.
- **Heap misconfiguration after pack updates.** Launcher-managed servers can silently ignore JVM-args files; heap must be set in the launcher's own config. A pack update that resets it to defaults produces exactly the slow-burn flap pattern.
- **A phantom 0% SLO tile with the server healthy** is not an outage: the monitor drops its `server_version` label when a ping fails, forking the series. The recording rules aggregate that label away; if a phantom reappears after monitor upgrades, re-check that aggregation survives.

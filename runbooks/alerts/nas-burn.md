# NasErrorBudgetBurnFast / NasErrorBudgetBurnSlow

| Alert | Severity | Condition |
|-------|----------|-----------|
| NasErrorBudgetBurnFast | critical | burn >14.4x on 1h+5m windows, 2m |
| NasErrorBudgetBurnSlow | warning | burn >6x on 6h+30m windows, 15m |

**Signal**: node_exporter scrape success on JBNAS01 (192.168.0.201). The NAS backs SMB shares, the Plex media library, and the backup target, so unreachability here cascades widely. Fast burn at critical is justified: this host going dark has historically meant a hung VM, not a blip.

## Triage

1. **Confirm scope**: can you SSH to the NAS? Does SMB answer from any client? A dead node_exporter with SSH fine is a monitoring-agent problem (restart the exporter unit), not an outage.
2. **Console, not network**: if unreachable, open the VM console from the hypervisor (Proxmox on JBSRV01, VM 101). A hung VM shows a console frozen mid-IO or kernel messages; note them before any reset.
3. **Check what the host was doing when it died**: on the hypervisor, look for storage-controller errors in `dmesg` around the failure time, and after recovery check `zpool history` for whether a scrub was running.

## Known causes from this estate's history

- **A single quiet drive can hang the whole storage controller.** A drive with SMART PASSED hung the LSI SAS HBA under sustained read (scrub load), freezing the VM. If crashes correlate with scrub timing, suspect a drive even when SMART looks clean.
- **Backup self-deadlock (fixed, but the pattern generalises)**: the hypervisor's backup job once backed the NAS VM up **onto the NAS's own NFS export**; the snapshot froze the VM serving the storage the backup was writing to. The NAS VM is excluded from that job now. If a hang coincides with the weekly backup window, check what the backup job targets before anything else.
- **IO throttling alerts alongside this one** usually mean bulk media ingest, which is benign; see [node-resource-pressure.md](node-resource-pressure.md) for the stall-vs-throughput discriminator.

## After recovery

- `zpool status -v` both pools; run/finish a scrub if the failure interrupted one.
- Check clients: SMB sessions can come back wedged ("connected", 0-byte reads); the fix is rebooting/remounting the **client** (see [plex-library-unreadable.md](plex-library-unreadable.md)).
- Verify the backup heartbeats resume ([backup-heartbeat-stale.md](backup-heartbeat-stale.md)).

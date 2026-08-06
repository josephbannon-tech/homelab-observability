# ZfsPoolNotOnline

| Severity | Condition |
|----------|-----------|
| critical | any pool except boot-pool reporting a non-ONLINE state, sustained 5m |

**Signal**: `node_zfs_zpool_state` from the NAS's node_exporter. DEGRADED means redundancy is reduced or gone; how bad that is depends entirely on the pool's geometry, so know your pools: a single-parity pool DEGRADED is running with **zero** remaining failure tolerance.

## Triage

1. **State and cause**: `zpool status -v <pool>` on the NAS. Read the per-device column: which member is FAULTED/UNAVAIL/REMOVED, and are there read/write/checksum error counts on others.
2. **Transient vs real**: a device that dropped off the bus (cable, HBA reset, warm-reseat) shows UNAVAIL with the pool otherwise clean. Check `dmesg -T` for link resets around the transition. After reseating a drive on a SAS HBA, rescan the SCSI host (`echo "- - -" > /sys/class/scsi_host/hostN/scan`) to re-enumerate without a reboot; then `zpool online <pool> <device>`.
3. **Real device failure**: replace and resilver (`zpool replace`). While the resilver runs the pool is at its most vulnerable and slowest; hold off bulk workloads until it completes.
4. **Checksum errors with devices ONLINE**: run a scrub and watch whether counts grow; growing checksum errors on one member is early device failure, spread across members points at controller/cabling/RAM.

## While degraded

- Stop non-essential write load to the pool.
- If the degraded pool has no remaining redundancy, prioritise copying anything irreplaceable off it **before** the resilver (a resilver is a sustained full-pool read, exactly the load that finishes marginal drives; see the quiet-HBA-hang history in [nas-burn.md](nas-burn.md)).
- Do not clear errors (`zpool clear`) until the underlying cause is identified; the counts are your evidence.

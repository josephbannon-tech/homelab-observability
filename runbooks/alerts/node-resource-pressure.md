# NodeMemoryThrottled / NodeIOThrottled

| Alert | Severity | Condition |
|-------|----------|-----------|
| NodeMemoryThrottled | warning | memory PSI full-stall rate >10% over 5m, sustained 15m |
| NodeIOThrottled | warning | IO PSI full-stall >10% AND local disk throughput <30 MB/s AND local disk busy >50%, sustained 15m |

**Signal**: kernel pressure-stall information, not utilisation. High RAM *usage* is healthy on a ZFS host (ARC), a JVM game server (fixed heap), and a hypervisor (guest allocation); PSI measures the fraction of time tasks were fully stalled, which is actual suffering. The IO alert additionally requires *low throughput on a busy disk*, which is the discriminator between a struggling device and legitimate bulk work.

## Memory: triage

1. `free -h` and `vmstat 1` on the instance: sustained non-zero `si`/`so` is swap thrash.
2. `ps aux --sort=-%mem | head` for the runaway; on a ZFS host confirm ARC is shrinking under pressure as designed before blaming it.
3. Chronic pressure without a runaway is genuine undersizing: grow the VM's allocation or move a workload.

## IO: triage, in this order

1. **Rule out memory pressure first**: heavy swap paging shows up as IO stall. If `vmstat` shows swapping, it's the memory runbook wearing a disguise.
2. **Rule out a benign bulk writer**: on the NAS, `smbstatus --locks` showing active writes into an inbox/staging path during a large ingest is normal operation; the interlock conditions try to exclude this (bulk ingest moves data fast), but slow-source downloads can still trip it.
3. **Then suspect a device**:
   - `dmesg -T | grep -iE 'ata|reset|timeout|error'` for link resets and command timeouts.
   - `zpool status` for read/write/checksum errors or a resilver in flight.
   - `zpool iostat -vl 1`: one member with disk-wait far above its siblings is your suspect.
4. **The quiet-failure case**: a drive can pass SMART completely and still hang or crawl under sustained load, taking the controller with it. Correlate stall onset with scrub schedules (`zpool history`), and treat a repeat offender as replaceable even with clean SMART.

## Interpreting the pair

Both firing together on the same host at the same time usually collapses into one cause: swap thrash (memory is the root) or a dying disk making everything wait (IO is the root). The `vmstat` swap columns are the tiebreaker.

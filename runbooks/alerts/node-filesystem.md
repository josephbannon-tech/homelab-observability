# NodeFilesystemLow / NodeFilesystemCritical

| Alert | Severity | Condition |
|-------|----------|-----------|
| NodeFilesystemLow | warning | below 15% free, sustained 1h |
| NodeFilesystemCritical | critical | below 5% free, sustained 15m |

**Signal**: `node_filesystem_avail_bytes / node_filesystem_size_bytes` per mounted filesystem, evaluated per-series across every host, with network filesystems (`cifs|nfs|nfs4|fuse.*|smb3|9p`) and pseudo-filesystems excluded. The exclusion is load-bearing: a client must not page on a server's free space, and a network mount can surface server-side constructs (a per-user quota, a staging inbox) that are not local capacity. This rule set exists because a host once filled its root FS to 98% with nothing watching.

## Triage

1. **Where is it going**: `df -h` on the instance, then `du -xh --max-depth=2 / | sort -h | tail -20` on the offending mountpoint (`-x` stays on one filesystem).
2. **Usual suspects by host type**:
   - Journald/log growth: `journalctl --disk-usage`, vacuum if the cap is misconfigured.
   - Package caches: apt archives, old kernels.
   - Container hosts: dead images/volumes (`docker system df` / `crictl`), and local-path PVs that grew (Prometheus/Loki retention on the cluster node).
   - Application data that should be on the NAS but landed locally.
3. **Critical (<5%)**: free space *now* before investigating root cause; a full root filesystem takes services down with it. Truncating a fat log beats a tidy investigation while writes are failing.
4. **Check the growth rate, not just the level**: the capacity dashboard's days-to-full projection distinguishes a slow creep (schedule cleanup) from an active runaway (find the writer now: `lsof +L1` also catches deleted-but-open files that `du` can't see).

## Prevent recurrence

If a cleanup was one-off, add or fix the periodic maintenance job for that host and confirm its success heartbeat is monitored, so the next creep pages the heartbeat alert before this one fires.

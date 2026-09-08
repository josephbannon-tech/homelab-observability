# MediaScanRunningTooLong / MediaScanWithoutJob

| Alert | Severity | Condition |
|-------|----------|-----------|
| MediaScanRunningTooLong | warning | `media_pipeline_scan_in_progress == 1` and the current scan started more than 6 h ago, sustained 5 m |
| MediaScanWithoutJob | warning | in-progress gauge set for more than 5 m with **zero** active `media-scan-*` Jobs on the cluster, sustained 10 m |

**Signal**: the verify-gated media ingest pipeline runs one decode scan at a time under a lock. The NAS-side orchestrator exports a gauge pair (`media_pipeline_scan_in_progress`, `media_pipeline_scan_start_timestamp_seconds`) from the node_exporter textfile collector while it waits on a Kubernetes Job that runs ffmpeg against the file. kube-state-metrics sees the Job. These two rules compare the two views.

A long scan is not by itself a fault: a three-pass decode of a feature-length 4K remux takes hours. What these alerts catch is the scan **outliving its Job**, which is what a stall looks like from outside.

## Why this exists

A scan pod was OOM-killed within a minute of starting. The Job went `Failed`, Kubernetes reaped it ten minutes later, and the orchestrator's `kubectl wait --for=condition=complete` sat there for seven hours, because that command only returns on the condition it was asked for. The pipeline held its lock the whole time, nothing else ingested, and no alert fired: the only heartbeat had a threshold sized for the slowest legitimate scan. The orchestrator now polls for either terminal condition, and these rules exist so that if any other blocking wait creeps in, it is visible within minutes.

## Operator loop

1. **Which alert?**
   - `MediaScanWithoutJob` is the incident shape. Go to step 2.
   - `MediaScanRunningTooLong` alone (Job still active) is probably a slow decode. Go to step 4.
2. **Confirm the orchestrator is blocked.** On the NAS:
   ```
   pgrep -af 'media-scan.sh|kubectl'
   journalctl -t media-scan -t media-pipeline --since '-8h' | tail
   ```
   A `kubectl` child older than the newest Job in the log, with no verdict line after `scanning:`, is the wedge.
3. **Unblock.** `kill` the `kubectl` child (never the pipeline's flock wrapper by hand: the orchestrator falls through, logs `ERROR: no verdict` / `scan ERROR`, and releases the lock on its own). Then find out why the Job died before the next cron tick retries it:
   ```
   kubectl -n monitoring describe job <name>          # if not yet reaped
   ```
   or in Loki: `{namespace="monitoring", pod=~"media-scan-.*"}` and the `kube_pod_container_status_terminated_reason` series for the pod. `OOMKilled` on a file that plays fine usually means the scanner is decoding a stream it should not (embedded cover art is a "video stream"; the scanner selects `-map 0:V` to exclude attached pictures for exactly this reason).
4. **Slow decode or hung decode?** On the cluster node:
   ```
   kubectl -n monitoring get jobs,pods | grep media-scan
   top -bn1 | grep ffmpeg
   ```
   ffmpeg at full CPU with a growing log is working; let it run, the Job's own `activeDeadlineSeconds` (12 h) is the backstop. ffmpeg at zero CPU in `D` state is blocked on the NFS read from the NAS: check the NAS pool (`zpool status -x`, load) before touching the pod.
5. **Clear.** Both alerts resolve on their own once the orchestrator writes the gauge back to 0, which it does at the end of the scan and at the start of every run.

## Trust boundaries

- **A scan that failed is not a corrupt file.** An ERROR verdict (Job failed, gone, or timed out) leaves the file in the inbox untouched and the pipeline retries it. Only a two-pass reproducible decode error at the same offset ever deletes anything.
- **The gauge is written by the orchestrator, not the Job.** If the NAS textfile collector is stale, the whole pair goes stale together and neither rule fires; the general backup-heartbeat rule covers the orchestrator's own liveness.
- **Pinned scanner image.** The ffmpeg image is pinned by digest. A behaviour change in the detector should come from a deliberate bump with a re-scan of a known-clean 4K file, never from a floating tag.

# Alert design

The estate runs ~35 curated alert rules alongside a pruned subset of kube-prometheus-stack's built-ins. The full rule definitions with inline rationale are in [homelab-gitops `charts/kube-prometheus-stack/values.yaml`](https://github.com/josephbannon-tech/homelab-gitops/blob/main/charts/kube-prometheus-stack/values.yaml). This document covers the curation decisions.

## Every alert carries the same annotation schema

Free-text `description` fields rot. Every curated rule instead carries a fixed schema, and the Discord notification template renders it as a structured diagnostic block:

| Annotation | Content |
|------------|---------|
| `summary` | One line, what is wrong |
| `host` | Which machine, with IP |
| `value` | The observed value, templated from `$value` |
| `threshold` | What it was compared against, including the `for:` duration |
| `impact` | Who or what is affected, in user terms |
| `action` | The first diagnostic command to run, copy-pasteable |
| `runbook_url` | Stable link to the matching runbook |

The `impact` and `action` fields are the ones that matter at 2am. An alert that says "The plex-exporter to Plex-API path is failing. This is NOT the streaming path, family playback may be perfectly fine" pre-answers the triage question that would otherwise cost ten minutes of dashboard-hopping. The template falls back gracefully to `description` and instance labels for built-in alerts that lack the schema.

## Culling the built-ins

kube-prometheus-stack ships 148 alert rules. On a single-node k3s cluster, a large fraction are dead weight or worse:

- **Dead**: `etcd` (k3s uses kine), `kubeProxy`/`kubeControllerManager`/`kubeScheduler` as separate components (k3s embeds them), `windows`.
- **Multi-node noise**: `kubernetesResources` overcommit alerts, `network`, quorum-shaped rules that assume replicas.
- **Actively harmful**: `nodeExporterAlerting` carried `NodeMemoryHighUtilization`, which false-fires on any host where high RAM use is the design (see below), and duplicated curated filesystem rules with different thresholds, guaranteeing double pages.

Eleven groups are disabled via `defaultRules.rules`; ~69 built-in alerts remain (workload health, Prometheus/Alertmanager self-monitoring). All built-in recording-rule groups stay enabled because dashboards depend on them. The principle: every firing alert must be actionable on this estate, and a rule that cannot fire truthfully here is deleted, not silenced.

## Memory pressure: PSI, not utilisation

A memory-utilisation alert (>90% used) false-fired on three hosts in its first week, all healthy: the NAS (ZFS ARC exists to consume RAM), the Minecraft server (a JVM with a fixed 10G heap), and the hypervisor (guest allocation). High utilisation is the correct operating state for all three.

The replacement alerts on kernel PSI (`node_pressure_memory_stalled_seconds_total`, and the IO equivalent): the fraction of wall-clock time tasks spent fully stalled on memory. That measures suffering rather than accounting, and it has not false-fired since. The same reasoning applies to `CPUThrottlingHigh`: cgroup throttling is independent of average CPU usage, and a bursty exporter can throttle hard while averaging single-digit percent (see [runbooks/tune-cpu-throttling-alert.md](../runbooks/tune-cpu-throttling-alert.md)).

## Filesystem alerts must exclude network mounts

The disk-space rules (`<15%` warn for 1h, `<5%` critical for 15m) originally evaluated every mounted filesystem. A client's CIFS mount of a NAS share then paged when a server-side quota filled during a routine automated transfer: normal operation on the server, a "disk critical" page on the client. A client must not page on a server's free space, and a network mount can surface server-side constructs (per-user quotas, staging buffers) that are not durable local capacity. The `fstype` matcher now excludes `cifs|nfs|nfs4|fuse.*|smb3|9p`; the NAS pool's own capacity is still alerted by the NAS's own node_exporter, where it belongs.

## SMART alerts: attributes, not exit codes

The first cut alerted on `smartctl_exit_status`, a bitfield that latches historical error-log entries, so one past event alerted forever. The replacement alerts on the attributes that predict failure: non-zero reallocated/pending/uncorrectable sector counts (warning), SSD life-left below 5% (warning), and the drive's own `smart_status` failing (critical). SMART is collected only on the hypervisor: the NAS VM sees virtio devices and `smartctl` returns garbage there, so the physical host scrapes all drives once.

## Known gap: alert fan-out from a single failure

A 33-minute outage of the Plex host once produced four pages: the genuine blackbox reachability alert plus three downstream kube alerts (`TargetDown`, `KubePodCrashLooping`, `KubeDeploymentReplicasMismatch`) from an exporter that crash-looped when its target vanished. The exporter was fixed to park-and-wait instead of crash-looping (an entrypoint gate that waits for the target before launching, chosen over an initContainer because init containers do not re-run on main-container restarts). Alertmanager inhibition rules to suppress the generic alerts while the service-level alert fires remain an open item, noted here because pretending the gap is closed would defeat the point of writing any of this down.

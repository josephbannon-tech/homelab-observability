# Tune a CPUThrottlingHigh alert

**Symptom**: `CPUThrottlingHigh` fires for a pod whose CPU usage looks trivially low.

**The mental model correction**: the alert fires on `container_cpu_throttled_seconds_total`, a cgroup throttle counter, which is independent of *average* CPU usage. CFS quota is enforced per 100ms period. A Python exporter that bursts on every scrape can exhaust its quota within many individual periods, and be throttled hard, while averaging single-digit percent of a core. Average-based intuition ("it's barely using CPU, the alert must be wrong") is exactly backwards: the alert is measuring the bursts the average hides.

**Diagnose**:

```bash
kubectl top pod -n <namespace>
```

If average usage is low while the alert fires, it's burst throttling, not runaway consumption.

**Fix**: raise `limits.cpu` on the container (in this estate, an exporter went `100m` → `500m`). Leave `requests.cpu` unchanged: requests drive scheduling and capacity planning, limits drive throttling, and this is purely a throttling problem.

**Expect resolution lag**. The rule's `for:` duration (15m here) must evaluate false continuously, then Alertmanager waits up to `resolve_timeout` (5m default) before sending resolved. Up to ~20 minutes after the fix is normal; don't re-triage a resolving alert.

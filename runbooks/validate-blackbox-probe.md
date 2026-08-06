# Validate a new or changed blackbox probe before merge

**When**: any PR that adds or changes a `Probe` CRD, a blackbox-exporter module, or the recording rules over `probe_success`.

**Why**: twice in a row, a new blackbox probe shipped silently broken. ArgoCD reported `Synced`, the pod was `Ready`, the Probe CRD existed, and `probe_success` was 0 from the first evaluation. A 30-day sliding-window SLO then stays poisoned for weeks after the fix. The two failure modes seen:

1. The DNS module queried a name with no record; the DNS server correctly answered `NXDOMAIN`, and `valid_rcodes: [NOERROR]` correctly rejected it. Probe, server, and config each did their job; the composition was broken. It went unnoticed for nine days, and the critical alert built on it selected on a label the metric never carried, so it evaluated an empty vector and could never have fired at all.
2. A misspelled module field crash-looped only the new ReplicaSet while the Service routed to the old pod (see [validate-chart-values-pre-merge.md](validate-chart-values-pre-merge.md)).

Both are invisible from `kubectl get application` and `kubectl get pods`. The pod and the SLI are independently failable.

## The check

Port-forward Prometheus and interrogate the real signal:

```bash
kubectl -n monitoring port-forward svc/kube-prometheus-stack-prometheus 9090:9090 >/dev/null 2>&1 &
PF=$!; sleep 2

# 1. The probe reaches the target and succeeds.
curl -sG http://127.0.0.1:9090/api/v1/query \
  --data-urlencode 'query=probe_success{job="<newjob>"}' | jq '.data.result'

# 2. The recording rule actually produces a series (label selectors match).
curl -sG http://127.0.0.1:9090/api/v1/query \
  --data-urlencode 'query=slo:<service>:availability:ratio_30d' | jq '.data.result'

# 3. No silent rejection in the exporter. The Service routes to whichever pod
#    is Ready, so a CrashLoopBackOff on the new ReplicaSet can hide behind the
#    old one. List all pods, not the Service.
kubectl -n monitoring get pods -l app.kubernetes.io/name=prometheus-blackbox-exporter
kubectl -n monitoring logs -l app.kubernetes.io/name=prometheus-blackbox-exporter --tail=20 --all-containers

kill $PF 2>/dev/null
```

**Expected**: (1) returns `value: [..., "1"]` for every target; (2) returns a non-empty result; (3) shows exactly one Ready pod per replica and no config errors in the logs.

If any check fails, the PR is not ready. Fix, force a refresh of both the exporter and the operator, and re-check:

```bash
kubectl -n monitoring rollout restart deploy/prometheus-blackbox-exporter
kubectl -n monitoring rollout restart deploy/kube-prometheus-stack-operator
```

## Design notes for the probe itself

- Probe a query that must succeed in production, not a synthetic name. A health-check domain that doesn't resolve is a permanent false negative.
- Pick target endpoints that do no heavy work, so the probe measures reachability rather than load (the reasoning is in [slo-design](../slo/slo-design.md)).
- If recording rules select on module or job labels, verify the label actually exists on the emitted metric. Probe-CRD metrics carry the job you configure on the Probe, not the module name.

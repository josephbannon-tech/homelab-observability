# Force a prometheus-operator scrape config reload

**Symptom**: you changed `additionalScrapeConfigs` in the chart values, ArgoCD shows Synced/Healthy, the rendered Secret content is correct, but the new scrape job never appears in the running Prometheus.

**Cause**: the operator does not reliably reconcile on Secret *content* changes; it watches the objects it manages, and a Secret whose name and shape are unchanged can slip through without a config regeneration.

**Fix**:

```bash
kubectl rollout restart deployment kube-prometheus-stack-operator -n monitoring
```

**Verify** the job reached the live config (the operator writes the final config into the Prometheus pod):

```bash
kubectl exec -n monitoring prometheus-kube-prometheus-stack-prometheus-0 -- \
  grep 'job_name: <newjob>' /etc/prometheus/config_out/prometheus.env.yaml
```

Then confirm the target is up in the Prometheus UI or via `/api/v1/targets`.

Note the general lesson, which recurs across this stack (Grafana admin password, image-renderer token, this): **anything that consumes a Secret via env or a generated file only reads it at process start or reconcile time**. GitOps applying the new Secret does not mean any running process has seen it. When a Secret-shaped change "didn't take", the question is always "which process is still holding the old value", and the answer is usually a restart away.

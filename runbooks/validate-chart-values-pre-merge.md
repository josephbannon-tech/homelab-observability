# Validate chart values before merge

**When**: any chart edit that touches a tool's own configuration schema: blackbox-exporter modules, Prometheus rules, Alertmanager routes, Grafana JSON.

**Why**: `yaml.safe_load` succeeding is necessary but nowhere near sufficient. Field names in tool configs parse as valid YAML even when misspelled; the binary only rejects them at process start. Helm doesn't catch it either, because Helm renders strings, not schemas.

The incident that produced this runbook: a blackbox module was committed with `fail_if_body_not_matched_regexp` instead of `fail_if_body_not_matches_regexp` (matched vs matches, one character of tense). The YAML was valid, ArgoCD synced clean, but blackbox-exporter rejected the config at startup and the new ReplicaSet crash-looped. The Service kept routing to the old pod, which didn't have the new module, so every probe against it returned HTTP 400 and the brand-new SLI read 0 while the monitored service was perfectly healthy.

## The gate

**1. Lint the syntax:**

```bash
python3 -c "import yaml; yaml.safe_load(open('charts/<chart>/values.yaml'))"
```

**2. Render the templates** (catches templating bugs):

```bash
helm template charts/<chart> | python3 -c "import yaml,sys; list(yaml.safe_load_all(sys.stdin))"
```

**3. Dry-load against the actual binary** when the change touches a tool config block:

- blackbox-exporter: extract the `config` block to a file, then
  `docker run --rm -v $(pwd)/blackbox.yml:/etc/blackbox_exporter/config.yml quay.io/prometheus/blackbox-exporter:<version> --config.check`
- Prometheus rules: `promtool check rules <rendered-rules>.yaml` on the rendered `PrometheusRule`'s `spec.groups` extracted to a file
- Alertmanager: `amtool check-config <rendered-config>.yaml`

Note on where to run promtool/amtool: recent Prometheus images are distroless (no shell, no tar), so `kubectl exec`-ing promtool with a copied file into the running pod no longer works. Use a local binary, a container as above, or validate expressions against the live query API (`/api/v1/query` returns a parse error for anything promtool would reject):

```bash
curl -sG http://<prometheus>/api/v1/query --data-urlencode 'query=<new expression>'
```

**4. After sync, check Pod readiness, not just app health.** A config-rejecting tool will show ArgoCD `Synced` while the new pod crash-loops and the old one keeps serving stale config:

```bash
kubectl -n monitoring get pods
kubectl -n monitoring logs <pod> --previous
```

The failure pattern to internalise: the deployment pipeline reported success at every layer it could see, and the only symptom was a wrong number from a new metric. If a change adds or alters a probe, continue to [validate-blackbox-probe.md](validate-blackbox-probe.md).

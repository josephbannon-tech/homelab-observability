# Trigger a Grafana dashboards reload by hand

**When**: dashboard ConfigMaps have synced but the boards haven't appeared or updated in Grafana.

**Background**: dashboards are provisioned as ConfigMaps (one per board, generated JSON) picked up by the `grafana-sc-dashboard` sidecar, which writes the JSON to disk and POSTs to Grafana's provisioning-reload API. If that POST gets a 401, the usual cause in this stack is the admin password desyncing from the SQLite DB after a chart upgrade (the chart hook regenerates the Secret; the DB on the PV keeps the old password).

**Manual reload**:

```bash
PW=$(kubectl get secret <grafana-admin-secret> -n monitoring -o jsonpath='{.data.admin-password}' | base64 -d)
kubectl exec -n monitoring deployment/kube-prometheus-stack-grafana -c grafana -- \
  wget -qO- --header="Authorization: Basic $(echo -n "admin:$PW" | base64)" \
  --post-data="" http://localhost:3000/api/admin/provisioning/dashboards/reload
```

**Gotcha that costs people an hour**: `wget` inside the Grafana container is BusyBox. It does not support `--user`/`--password`, and fails in a way that reads like an auth problem. Build the Basic header by hand, as above.

**If the reload itself returns 401**: the password is desynced. Reset the DB password to match the Secret first:

```bash
kubectl exec -n monitoring deployment/kube-prometheus-stack-grafana -c grafana -- \
  grafana cli admin reset-admin-password "$PW"
```

`grafana cli` writes the DB directly and is not subject to login rate limiting; if repeated interactive 401s have tripped brute-force protection, restart the pod first, then reset. The durable fix for the desync class is env-var-based admin credentials so Grafana reads the Secret at startup instead of persisting a copy in SQLite; until then, this runbook is the bridge.

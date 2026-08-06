# Add metrics from a new host

**When**: a new VM, LXC, or physical host joins the estate.

1. **Install node_exporter** on the host, running as a dedicated system user under systemd. Platform quirks to expect: appliance OSes with read-only root filesystems need the binary on a data volume rather than `/usr/local/bin`; minimal LXCs may lack a `sudo`/`adm` baseline.
2. **Open the firewall** for the Prometheus host to reach `:9100`. Scope the rule to the scraper's source IP rather than the whole LAN; host metrics leak more than people expect (process names, mount paths, network peers).
3. **Add the target** to the `node-exporter-external` static scrape job in `charts/kube-prometheus-stack/values.yaml` ([homelab-gitops](https://github.com/josephbannon-tech/homelab-gitops)), with a `hostname=<HOST>` label. External hosts are deliberately static config, not ServiceMonitors: ServiceMonitor only applies to targets with an in-cluster Service.
4. **Push to main.** ArgoCD syncs the values.
5. **Verify the target appears** in Prometheus (Status → Targets, or `/api/v1/targets`). If it hasn't appeared within 5 minutes, the operator missed the Secret change; see [force-prometheus-operator-reload.md](force-prometheus-operator-reload.md).

**Done when**: `up{job="node-exporter-external", hostname="<HOST>"}` returns 1, and the host shows up in the per-host drill-down dashboard's hostname variable.

Follow with [add-host-logs.md](add-host-logs.md) so metrics and logs land together; a host that ships one but not the other is drift that surfaces at the worst time.

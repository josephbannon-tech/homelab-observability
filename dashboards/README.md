# Dashboards

Seven boards covering an 8-host estate. All are **generated** by a single Python script ([`gen-dashboards.py` in homelab-gitops](https://github.com/josephbannon-tech/homelab-gitops/blob/main/charts/dashboards/gen-dashboards.py)) rather than hand-edited in the UI: panel helpers, shared threshold constants, and hoisted host/IP dicts, emitted as ConfigMaps for Grafana's provisioning sidecar. Hand-exported dashboard JSON is write-only; a generator gives readable diffs, consistent styling, and made this extraction mechanical.

The JSON files here are the same boards unwrapped from their ConfigMaps into plain importable Grafana JSON.

## Importing

Grafana → Dashboards → Import → upload a JSON file. The boards expect three datasources: a Prometheus (default), a Loki named `loki`, and (for trace correlation) a Tempo. Panels pin Loki explicitly at panel level, so a differently-named Loki datasource needs a search-and-replace on the `datasource` fields or a matching provisioned UID.

Queries reference this estate's label conventions: external hosts carry `hostname=<NAME>` on the `node-exporter-external` job, service jobs are `pihole`, `plex`, `minecraft`, `tautulli`, `blackbox-*`, and SLO panels read `slo:*` recording rules (definitions in [homelab-gitops values.yaml](https://github.com/josephbannon-tech/homelab-gitops/blob/main/charts/kube-prometheus-stack/values.yaml)). Importing onto a different estate means adapting those selectors; the recording rules are the part worth taking wholesale.

Every board opens with a firing-alerts strip scoped to that board's services, so "is anything wrong *here*" is answered before any chart is read. All panels carry `description` tooltips stating what the panel measures and what normal looks like.

## The boards

| Board | What it answers |
|-------|-----------------|
| [`family-services-slo`](family-services-slo.json) | Are the user-facing services meeting their SLOs? 30-day compliance tiles against target lines, per-service burn rates, then per-service detail rows including real playback/session telemetry ([screenshot](screenshots/family-services-slo.png)) |
| [`alerts-overview`](alerts-overview.json) | What is firing right now, and what has been firing lately, by severity ([screenshot](screenshots/alerts-overview.png)) |
| [`proxmox-overview`](proxmox-overview.json) | Hypervisor health: host CPU/memory/load, per-guest CPU/memory/disk/network via the Proxmox API exporter ([screenshot](screenshots/proxmox-overview.png)) |
| [`nas-zfs`](nas-zfs.json) | ZFS pool state, dataset fill against the datasets that actually hold data (not pool roots), ARC size and hit rate ([screenshot](screenshots/nas-zfs.png)) |
| [`capacity-backup-dr`](capacity-backup-dr.json) | Fleet-wide free space with inverted (low=red) thresholds, days-to-full linear projections, backup last-success heartbeats with cadence-aware thresholds, SMART wear ([screenshot](screenshots/capacity-backup-dr.png)) |
| [`logs-network-dns`](logs-network-dns.json) | Loki log/error/warning rates per host, noisiest systemd units, SSH auth failures, live Pi-hole DNS stats and the blackbox DNS SLI ([screenshot](screenshots/logs-network-dns.png)) |
| [`per-host-fleet`](per-host-fleet.json) | Any-host drill-down behind a `$hostname` variable: CPU by mode, memory breakdown, disk/network IO, filesystem fill, journal noise ([screenshot](screenshots/per-host-fleet.png)) |

## Design notes that came from review

- **Bargauge thresholds must match direction.** Free-space and days-to-full gauges shipped with the default high=red thresholds, rendering a nearly-full filesystem green. The generator grew a `thresholds=` argument; low=red everywhere it belongs.
- **Query the dataset that holds the data.** The ZFS usage board originally queried pool-root datasets, which read near-zero because data lives in children.
- **State tiles render words, not numbers.** A value-map layer turns `0/1` into `UP/DOWN`, `ONLINE/DEGRADED`, `OK/FAIL`; a wall of bare booleans is unreadable at a glance.
- **Kill redundant panels.** Anything duplicating another board's panel, or restating a chart as a bargauge, was cut in review. 100+ panels is already at the edge of maintainable.

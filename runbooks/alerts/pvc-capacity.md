# PersistentVolumeUsageHigh / PersistentVolumeFillingUp

| Alert | Severity | Condition |
|-------|----------|-----------|
| PersistentVolumeUsageHigh | warning | `kubelet_volume_stats` used/capacity > 80% for 30m |
| PersistentVolumeFillingUp | warning | `predict_linear` over 6h reaches zero within 4 days, already > 50% used, for 1h |

**Signal**: kubelet's per-PVC filesystem stats. **Scope caveat that matters for triage**: for `local-path` PVCs (a bind mount into the node's data disk, no quota) kubelet reports the *node disk's* fill under each PVC's name, so every local-path PVC reads the same percentage and the alert duplicates `NodeFilesystemLow` with a PVC label attached. Only NFS-backed claims (the Immich library) report their own export's capacity.

## Triage

1. **Which class?** `kubectl -n <ns> get pvc <name>`: `local-path` means read on; `nfs-static` means the NAS export is filling.
2. **local-path**: `df -h /var/lib/rancher/k3s/storage` on the node. Biggest consumers in order of likelihood: Loki chunks (90-day retention), Tempo, Prometheus TSDB, Immich thumbnails/encoded video after an import. `du -sh /var/lib/rancher/k3s/storage/*` names them.
3. **NFS (Immich originals)**: `zfs list` on the NAS for the dataset; pause an import if one is running; the dataset has no quota by design, the pool does.
4. **FillingUp without UsageHigh**: a bulk write is in progress (import, log burst). Decide whether to let it finish; the 4-day horizon is there to make that a decision rather than a 03:00 page.

## Fix patterns

- Retention is the first lever (Loki/Tempo values in the gitops repo), the PVC size the second (local-path sizes are nominal, the disk is the limit), the disk the last.

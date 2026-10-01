# RadicaleDown

| Alert | Severity | Condition |
|-------|----------|-----------|
| RadicaleDown | warning | `probe_success{job="blackbox-radicale"} == 0` for 10m |

**Signal**: a blackbox HTTP probe of Radicale's public static page on the NodePort, the exact path a phone takes. Phones sync every 15 minutes and keep a local copy, so 10 minutes is a real outage, not a blip; edits made in the meantime queue on the device.

**Probe quirk**: Radicale's built-in server answers HTTP/1.0, which the default `http_2xx` module rejects (`Invalid HTTP version number`, status 200, `probe_success 0`). The probe uses its own module with `HTTP/1.0` in `valid_http_versions`. If someone "simplifies" it back to `http_2xx`, this alert fires with the service healthy.

## Triage

1. `kubectl -n pim get pods,svc` and `curl -sI http://<node>:30232/.web/` from a LAN host. 200 = probe or NodePort problem; connection refused = pod or Service.
2. Pod logs: `kubectl -n pim logs deploy/radicale --tail=50`. Auth errors are per-user and do not affect the probe.
3. Storage: the collections tree is on a local-path PVC. If the node's data disk filled, `PersistentVolumeUsageHigh` will be firing too.
4. If the pod is healthy but the probe fails on HTTP version, see the quirk above.

## Restore

Nightly archive on the NAS with a heartbeat; restore runbook in [homelab-runbooks](https://github.com/josephbannon-tech/homelab-runbooks/blob/main/restore/radicale.md).

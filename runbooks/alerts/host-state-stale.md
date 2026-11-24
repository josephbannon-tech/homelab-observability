# HostStateStale

| Severity | Condition |
|----------|-----------|
| warning | host-state file older than 30m, adlist file older than 3h, or a scraped host that has never written one; sustained 15m |

**Signal**: the host-state script stamps `homelab_host_state_last_run_timestamp_seconds` on every run; the Pi-hole adlist script stamps `pihole_adlist_last_run_timestamp_seconds`. A textfile metric from a script that has stopped is re-exported unchanged forever, so without this rule a dead timer would freeze "no failed units, no reboot needed" in place. The third branch of the expression fires for any host that node_exporter is scraped on and that has no host-state series, which makes enrolment the default: a new guest alerts until the script is installed.

While this alert fires, [HostRebootPending](host-reboot-pending.md), [SystemdUnitFailed](systemd-unit-failed.md), [PromtailNotShipping](promtail-not-shipping.md) and [PiholeAdlistStale](pihole-adlist-stale.md) are blind for that host.

## Triage

1. On the host: `systemctl status host-state-textfile.timer host-state-textfile.service`. On the storage appliance the job is a cron entry managed by the appliance, because units under `/etc` do not survive its upgrades.
2. Run the script by hand and read the error. The usual one is a missing or unwritable textfile directory.
3. `curl -s localhost:9100/metrics | grep node_textfile_scrape_error` must be 0. A non-zero value means a malformed `.prom` file in the directory, possibly from another job.
4. New host: install the script and timer. See [add-host-metrics.md](../add-host-metrics.md).
5. Retired host or job: delete its `.prom` file, or the stale series never goes away.

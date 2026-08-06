# Add log shipping from a new host

**When**: a new host joins the estate, normally right after [add-host-metrics.md](add-host-metrics.md).

**The two failure modes this runbook exists for.** Promtail needs two group memberships, and they fail differently:

- Without `systemd-journal`, the journal scrape produces **zero entries silently**. Promtail logs no error; `promtail_sent_entries_total` just stays at 0. Two hosts in this estate shipped nothing for a day before this was caught, which is why step 5 below is the real install verification.
- Without `adm`, the `/var/log` scrape fails **loudly** on any host with rsyslog (`permission denied` on `/var/log/syslog` every 10s), because Debian/Ubuntu ship those files as `0640 root:adm`.

## Procedure

1. **Install the Promtail binary**, pinned to the fleet's version so config behaviour is uniform.
2. **Create the user with both groups**:

   ```bash
   useradd --system --no-create-home --shell /usr/sbin/nologin promtail
   usermod -a -G systemd-journal,adm promtail
   install -d -m 0750 -o promtail -g promtail /var/lib/promtail
   install -d -m 0755 -o root -g root /etc/promtail
   ```

3. **Write `/etc/promtail/config.yaml`**, substituting the hostname (twice):

   ```yaml
   server:
     http_listen_port: 9080
     grpc_listen_port: 0
   positions:
     filename: /var/lib/promtail/positions.yaml
   clients:
     - url: http://192.168.0.207:31100/loki/api/v1/push
   scrape_configs:
     - job_name: journal
       journal:
         max_age: 12h
         labels:
           job: systemd-journal
           hostname: <HOST>
       relabel_configs:
         - source_labels: ['__journal__systemd_unit']
           target_label: unit
     - job_name: varlogs
       static_configs:
         - targets: [localhost]
           labels:
             job: varlogs
             hostname: <HOST>
             __path__: /var/log/{syslog,auth.log}
   ```

   The `varlogs` job is a no-op on hosts without rsyslog (modern Debian/Ubuntu don't install it by default); it is kept in the canonical config for fleet consistency, and does real work on hosts where rsyslog restores a text-log forensics window past a capped journald.

4. **Install and start the systemd unit** with `User=promtail`.
5. **Verify entries are actually shipping.** This is the canary that catches the silent failure mode:

   ```bash
   curl -s http://<HOST>:9080/metrics | grep ^promtail_sent_entries_total
   # must be > 0 within seconds (max_age: 12h replays recent journal on start)
   ```

6. **Verify the host reaches Loki**:

   ```bash
   curl -s http://192.168.0.207:31100/loki/api/v1/label/hostname/values
   ```

## Notes

- A high-churn host (frequent SSH sessions) benefits from a relabel collapsing `session-NNN.scope` unit names to one value, or the `unit` label's cardinality grows without bound.
- Appliance quirk seen in this estate: on TrueNAS the config lives on the data pool (read-only root FS), no dedicated user is created, and the systemd unit must be re-created after major upgrades because the OS dataset is replaced.

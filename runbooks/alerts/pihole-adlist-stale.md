# PiholeAdlistStale

| Severity | Condition |
|----------|-----------|
| warning | an enabled adlist at status 3 or 4 for every sample in 7 days, sustained 1h |

**Signal**: an hourly script on the DNS host reads Pi-hole's `adlist` table and writes `pihole_adlist_status{id,name}` and `pihole_adlist_entries{id,name}` for each enabled list. Status is Pi-hole's own: 1 downloaded, 2 unchanged upstream, 3 unavailable and served from the local cache, 4 unavailable with no cache. The status changes when gravity runs (weekly by default), so seven days of bad samples means about two failed runs in a row.

**Why this exists**: a blocklist that fails to download fails open and silent. Pi-hole keeps serving the last good copy from its cache, DNS answers, the block rate barely moves and every probe stays green. In 2026 five of seven lists pointed at URLs the upstream project had removed. They were served from cache for four months and surfaced only when a Pi-hole update emptied the cache. Backtested against that history, this rule fires in the first week of August.

## Triage

1. Open the list URL from the alert's `name` label in a browser or with `curl -I`. A 404 means the list moved or was retired; a timeout may be a passing upstream outage.
2. Moved: find the new URL in the upstream repository, check which format it serves (hosts, plain domains, or ABP-style), and update the list address in Pi-hole. ABP-style entries also cover subdomains; a plain-domain list matches exactly, so the same list name in a different format can block far less.
3. Run `pihole -g` and confirm the list's status returns to 1 and its entry count is in the expected range (`pihole_adlist_entries`).
4. Retired with no successor: disable the list so it stops alerting, and note the loss of coverage.
5. For a day afterwards, watch for names that were answered before the change and are blocked now. A repointed list is a different list.

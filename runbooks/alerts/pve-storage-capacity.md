# PveStorageAllocationHigh / PveStorageAllocationCritical

| Alert | Severity | Condition |
|-------|----------|-----------|
| PveStorageAllocationHigh | warning | above 75% allocated, sustained 1h |
| PveStorageAllocationCritical | critical | above 85% allocated, sustained 15m |

**Signal**: `pve_disk_usage_bytes / pve_disk_size_bytes` per Proxmox storage, from PVE Exporter. This is hypervisor-side allocation and is **not** covered by `NodeFilesystemLow`, which watches filesystems *inside* hosts and cannot see a Proxmox storage at all.

The thresholds are lower than a filesystem alert's on purpose. Two of the storages (`local-lvm`, `local-lvm-m2`) are **LVM-thin pools**, where full is categorically worse than a full filesystem:

- Guests are over-provisioned against the pool, so the pool can fill while every guest still believes it has free space.
- At 100% allocation **every guest backed by that pool takes I/O errors simultaneously**.
- A guest cannot recover itself. Deleting files inside a guest returns nothing to the pool unless the discard actually reaches it.

So this alert has to page well before the cliff, not at it.

## Why this rule exists

On 2026-08-08 `local-lvm` was found at **87.83% allocated**, with 198G provisioned into a 195G pool, climbing on its own and with `VGFree` at 0 so there was no escape hatch. Nothing was watching it; it was found by accident while answering an unrelated capacity question. Remediation recovered ~94 GB and took it to 41.82%.

## Triage

1. **Identify the storage and its type.** `pvesm status` on JBSRV01. `lvmthin` entries are the dangerous ones; `dir` and `nfs` are ordinary capacity.
2. **For a thin pool, get the real numbers**: `lvs -o lv_name,lv_size,data_percent,metadata_percent`. Watch `metadata_percent` too — exhausting thin *metadata* wedges the pool just as hard as exhausting data, and it is easy to miss.
3. **Find which guests are consuming it**: `lvs -o lv_name,data_percent | sort -k2 -n`. Thin volumes are named `vm-<vmid>-disk-N`.
4. **Check discard is actually configured** on each offender:
   ```
   qm config <vmid> | grep scsi0     # want discard=on,ssd=1
   ```
   Without `discard=on`, QEMU accepts the guest's SCSI UNMAP and silently drops it. The volume then grows to 100% of its provisioned size and never shrinks, no matter how much the guest deletes or how often `fstrim` runs inside it.
5. **Reclaim**:
   - VM with discard already on: `fstrim -av` inside the guest.
   - LXC: `pct fstrim <vmid>` from the host. `fstrim.timer` inside a container never runs (`ConditionVirtualization=!container`), by design.
   - VM missing discard: `qm set <vmid> --scsi0 <existing-volume>,discard=on,ssd=1` then a stop/start. **`qm set` drops any option you omit**, so copy the existing option string verbatim and add to it.
6. **Re-measure with `lvs -o data_percent`, not with fstrim's output.** `fstrim` reports the free ranges it issued discards for, not the blocks the backing store released. It will happily report gigabytes trimmed while the pool moves 0.5%.

## Gotchas

- **`fstrim.timer` being enabled proves nothing.** A guest can trim weekly for months and still sit at 94% if `discard=on` is missing at the hypervisor. That is exactly what happened here.
- **`Persistent=yes` does not trim at boot.** It only catches up when the timer's window was actually missed while the machine was down, so a reboot does not trigger a reclaim.
- **Only `virtio-scsi-single` supports `iothread`.** Guests on `virtio-scsi-pci` must not have it added while you are editing the disk string.
- **Declare it in IaC.** If the fleet is under OpenTofu, `discard`/`ssd` must be in the HCL or the next apply reverts the fix silently.

## Prevent recurrence

The durable fix is discard on every thin-provisioned disk *plus* the declaration in IaC, not a one-off reclaim. If this fires for a `dir` or `nfs` storage instead, treat it as ordinary capacity: check backup retention (`prune-backups` on the storage) before adding space.

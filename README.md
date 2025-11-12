---
title: Proxmox Cluster — Laptop Stack
tags: [homelab, proxmox, zfs, vlan, pbs, automation]
last_updated: 2025-11-12
---

<!--
This template is GitHub-friendly Markdown (works in a repo README or MkDocs page).
Fill the placeholders in <angle brackets>. Remove sections you don’t need.
-->

# Proxmox Cluster — <Your Cluster Name>

**TL;DR:** 4-node Proxmox VE 8.x on repurposed laptops. Local **LVM-thin** on single SSD per node, flat network on `vmbr0` (`10.0.0.0/24`). Weekly **vzdump** to `NAS01` (Sun 03:00), retention **keep monthly: 2**, compression **ZSTD**. **Live migration:** QEMU VMs can migrate online without shared storage (disk is mirrored over the tunnel); expect longer migrations on 1G links. LXC typically needs downtime without shared storage.

<!-- expected hero image: cluster-hero.jpg -->
![Cluster overview](../assets/proxmox/cluster-hero.jpg)

!!! note
    Network is **flat (no VLANs)**. VLAN sections are omitted on purpose.

---

## At a Glance

| Nodes | CPU/RAM (total)                           | Storage (raw)                   | Networks                          | Backups                                    | HA                     |
|-----:|--------------------------------------------|---------------------------------|-----------------------------------|--------------------------------------------|------------------------|
| 4    | i5-6300U ×2, i5-3337U ×2 / **40 GB** RAM   | 256+256+128+128 GB ≈ **768 GB** | `vmbr0` flat (`10.0.0.0/24`)      | vzdump → **NAS01**, Sun 03:00, keep monthly: 2, ZSTD | No (manual migrations) |

**Use cases:** Mealie, Home Assistant, ATAK server, Ubuntu Jump Station, SmokePing, Zabbix, WireGuard, Wazuh, Omada SDN Controller, Chicken Coop Zigbee (MQTT), n8n.

---

## Cluster Health (quick checks)

- [ ] Quorum OK: `pvecm status` shows votes from all nodes
- [ ] Thin pool headroom on each node (`local-lvm`): >20% free
- [ ] Backups succeeded in last 7 days (vzdump log / NAS)
- [ ] CPU type consistent (e.g., `x86-64-v2-AES`) for portability
- [ ] Start/Shutdown order set on critical VMs; “On boot” = ✓

---

## Bridge (Flat) Overview

| Bridge | Uplink NIC (example) | Host IP (per-node)      | VLANs | VLAN-aware        | MTU  | Purpose                 | Notes                                  |
|------:|------------------------|--------------------------|-------|-------------------|------|-------------------------|----------------------------------------|
| vmbr0 | `<eth0 / enpXsY>`     | see **Inventory** below  | none  | **No (by design)**| 1500 | Management + VM network | VLAN-aware not required at present     |

---

## Inventory

### Nodes

| Node  | IP         | Cores | CPU Model                                      | RAM (GB) | SSD (GB) | NICs              | Role        | Notes                                  | On AC (Y/N) |
|:-----:|------------|------:|-----------------------------------------------|---------:|---------:|-------------------|------------|----------------------------------------|:-----------:|
| PVE01 | 10.0.0.140 | 4     | Intel(R) Core(TM) i5-6300U @ 2.40 GHz         | 16       | 256      | `<1G / USB 2.5G>` | Hypervisor | Local **LVM** on single SSD            | Y           |
| PVE02 | 10.0.0.141 | 4     | Intel(R) Core(TM) i5-6300U @ 2.40 GHz         | 16       | 256      | `<1G / USB 2.5G>` | Hypervisor | Local **LVM** on single SSD            | Y           |
| PVE03 | 10.0.0.142 | 4     | Intel(R) Core(TM) i5-3337U @ 1.80 GHz         | 4        | 128      | `<1G / USB 2.5G>` | Hypervisor | Local **LVM** on single SSD            | Y           |
| PVE04 | 10.0.0.144 | 4     | Intel(R) Core(TM) i5-3337U @ 1.80 GHz         | 4        | 128      | `<1G / USB 2.5G>` | Hypervisor | Local **LVM** on single SSD            | Y           |

### VMs / Services (Summary)

| Node | IP         | Name     | Service / Purpose          |
|:----:|------------|----------|----------------------------|
| TBD  | 10.0.0.127 | CTMEL01  | Mealie.io                  |
| TBD  | 10.0.0.128 | VMHA01   | HomeAssistant VM           |
| TBD  | 10.0.0.129 | VMTAK01  | ATAK Server                |
| TBD  | 10.0.0.130 | VMJS01   | Ubuntu Jump Station        |
| TBD  | 10.0.0.132 | CTSP01   | Smoke Ping Server          |
| TBD  | 10.0.0.133 | VMZAB01  | Zabbix                     |
| TBD  | 10.0.0.134 | VMWG01   | Wireguard                  |
| TBD  | 10.0.0.135 | VMWAZ01  | Wazuh                      |
| TBD  | 10.0.0.136 | CTOMD01  | Omada SDN Controller       |
| TBD  | 10.0.0.137 | PIMQTT01 | Chicken Coop Zigbee Device |
| TBD  | 10.0.0.138 | VMN8N    | N8N Server                 |

---

## Storage & Backups

### Local Storage Layout (per node)

> Example from **PVE01** (others similar; sizes vary).

| Node  | Disk Model (example)         | Size   | Partitioning                       | Volume Group | Datastore            | Notes                                        |
|-------|-------------------------------|--------|------------------------------------|--------------|----------------------|----------------------------------------------|
| PVE01 | Micron_1100_SATA_256GB       | 256 GB | BIOS boot, EFI, **LVM PV**         | `pve`        | `local-lvm` (thin)   | Serial: `18251D1A16A3` · Wearout: `19%`      |
| PVE02 | Micron_1100_SATA_256GB       | 256 GB | BIOS boot, EFI, **LVM PV**         | `pve`        | `local-lvm` (thin)   | Serial: `18221CCE1A3F`                       |
| PVE03 | SanDisk_SDSG2128G1052E       | 128 GB | BIOS boot, EFI, **LVM PV**         | `pve`        | `local-lvm` (thin)   | Serial: `134375402069`                       |
| PVE04 | TOSHIBA_THNSNJ128GDNJ        | 128 GB | BIOS boot, EFI, **LVM PV**         | `pve`        | `local-lvm` (thin)   | Serial: `949B303KDD3X`                       |

**Guardrail:** Alert if `local-lvm` (thin pool) free **<20%** to avoid “out of space” VM pauses.

### Backup Jobs (vzdump)

> Current Proxmox **Backup Job** (GUI):  
> Storage: **NAS01** · Schedule: **Sun 03:00** · Selection: **Include selected VMs** · Compression: **ZSTD (fast and good)** · Mode: **Stop** · Enabled: **Yes**  
> Retention: **Keep Monthly = 2** (others blank)

| Job Name      | Type   | Storage | Schedule   | Selection             | Compression | Mode | Enabled | Retention                 |
|---------------|--------|---------|------------|-----------------------|-------------|------|---------|---------------------------|
| `weekly-main` | vzdump | NAS01   | Sun 03:00  | Include selected VMs  | ZSTD        | Stop | ✓       | Monthly: 2 (others unset) |

**Restore tests (quarterly):** Restore one representative VM to an isolated network (or with NIC disconnected), validate service, record time-to-recover; update “Backups” table’s “Last Verify”.

**CLI equivalent (example)**
```bash
# Create/replace a weekly vzdump job (adjust VMIDs)
echo '0 3 * * 0 root vzdump 100 101 102 --storage NAS01 --compress zstd --mode stop' > /etc/pve/vzdump.cron
# Retention for NAS01 can also be set on the storage target or managed manually.

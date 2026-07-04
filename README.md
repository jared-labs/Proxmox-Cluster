# Proxmox VE Cluster: 6-Node Laptop Stack

## Overview

This document summarizes the architecture of a 6-node Proxmox VE 8.x cluster built from repurposed laptops and desktops. It is written for a portfolio audience, focusing on cluster design, storage strategy, networking, and the operational tradeoffs of running a real virtualization cluster on consumer hardware.

The cluster hosts 14+ VMs and containers providing security monitoring, vulnerability scanning, log aggregation, threat intelligence, automation, and IoT/home services across a flat 10.0.0.0/24 network.

## Environment

| Field | Value |
|-------|-------|
| Cluster Name | pve-cluster-01 |
| Nodes | 6 |
| Total CPU | i5-6300U ×2, i5-3337U ×2, i7-7500U ×1, FX-8350 ×1 |
| Total RAM | 88 GB |
| Total Storage (raw) | ~1.78 TB |
| Network | vmbr0 flat (10.0.0.0/24) |
| Backups | vzdump → NAS01, weekly, ZSTD, retain 2 monthly |
| HA | No (manual migrations) |
| Live Migration | QEMU online (disk mirrored over tunnel) |

## Architecture

```text
         10.0.0.0/24 Flat Network (vmbr0)
         ════════════════════════════════════
              │         │         │         │         │         │
         ┌────┴───┐┌────┴───┐┌────┴───┐┌────┴───┐┌────┴───┐┌────┴───┐
         │ PVE01  ││ PVE02  ││ PVE03  ││ PVE04  ││ PVE05  ││ PVE06  │
         │ i5-6300││ i5-6300││ i5-3337││ i5-3337││ i7-7500││ FX-8350│
         │ 16 GB  ││ 16 GB  ││  4 GB  ││  4 GB  ││ 16 GB  ││ 32 GB  │
         │ 256 GB ││ 256 GB ││ 128 GB ││ 128 GB ││ 512 GB ││ 500 GB │
         └────────┘└────────┘└────────┘└────────┘└────────┘└────────┘
              │         │         │         │         │         │
              └─────────┴─────────┴────┬────┴─────────┴─────────┘
                                       │
                                       ▼
                              ┌─────────────────┐
                              │  NAS01 (TrueNAS)│
                              │  Backup target  │
                              │  vzdump storage │
                              └─────────────────┘
```

Each node has local LVM-thin storage on a single SSD. No shared storage (no Ceph, no NFS for VM disks).

## Workload Distribution

| Service Category | Examples | Typical Placement |
|-----------------|----------|-------------------|
| Security & Monitoring | Graylog, Cribl, OpenVAS, MISP | PVE05, PVE06 (more RAM) |
| Automation | Ansible, n8n, Discovery | PVE07 (containers) |
| IoT & Home | Home Assistant, Mealie, OctoPrint | PVE01-04 (lighter workloads) |
| Networking | Omada Controller, WireGuard, HAProxy | Distributed |
| Situational Awareness | TAK Server | PVE05 (8 GB RAM VM) |

## Design Decisions

- **No shared storage:** Each node owns its local disk. This simplifies hardware (no SAN, no 10G requirement) but means live migration mirrors disk over the cluster network tunnel. For a home lab, the tradeoff is acceptable.
- **Flat networking (no VLANs):** A single bridge (`vmbr0`) on 10.0.0.0/24 keeps things simple while the lab is in a single physical location. VLAN segmentation is a future goal.
- **Consumer hardware:** Repurposed laptops with built-in batteries (UPS built-in), low power draw, and quiet operation. The FX-8350 desktop adds raw compute for heavier workloads.
- **No HA, manual migrations:** With no shared storage, automatic HA failover isn't practical. Manual migration is sufficient for a home lab where downtime tolerance is high.
- **CPU type `x86-64-v2-AES` for portability:** VMs use this CPU type for migration compatibility across the mixed Intel/AMD fleet. Specific VMs that need AVX (MongoDB) use `cpu: host` and sacrifice migration.
- **ZSTD backup compression:** Fast compression with good ratios. Weekly full backups to NAS with 2-month retention balances storage use against recovery options.

## Quirks and Gotchas

- 1 GbE migration speed: Expect longer live migrations on 1G links since disk is mirrored over the tunnel.
- LXC without shared storage: Containers typically need downtime for migration.
- CPU type consistency: `x86-64-v2-AES` for portability, `host` only when AVX is required.
- Thin pool headroom: Keep >20% free on `local-lvm` to prevent VM pauses from thin-pool exhaustion.
- Laptop lids: Must be configured to not suspend on lid close (`HandleLidSwitch=ignore` in logind.conf).

## What This Demonstrates

This cluster shows practical virtualization engineering at home-lab scale: capacity planning across heterogeneous hardware, storage tradeoffs without enterprise SAN, backup strategy, migration capabilities, and workload placement decisions. The cluster runs real services 24/7 with monitoring, patching, and operational procedures built around its constraints.

---

Sanitized for public portfolio use.

For the node inventory, storage layout, backup configuration, and quick reference commands, see [OPERATIONS.md](./OPERATIONS.md).
For adding new nodes to the cluster, see the [New-Proxmox-Node-Runbook](https://github.com/jared-labs/New-Proxmox-Node-Runbook) repo.

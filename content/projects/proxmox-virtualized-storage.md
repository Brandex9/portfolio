---
title: "Multi-Node Hypervisor Fabric & Hardware-Accelerated Storage Cluster"
date: 2026-09-30
draft: false
tags:
  ["Proxmox", "Unraid", "ZFS", "PCIe-Passthrough", "10GbE", "Virtualization"]
summary: "Implementation of a 3-node fault-tolerant virtualization architecture featuring IOMMU hardware passthrough, local ZFS mirror pools, a 10G MikroTik spine, and decoupled power resiliency."
weight: 2
---

## Architecture Overview

A purpose-built hybrid hypervisor architecture running Proxmox VE, combining high-efficiency edge compute, high-throughput array storage, a dedicated 10G SFP+ switching fabric, and out-of-band power orchestration.

```text
                [ ISP Fiber Drop (SC/APC Optical In) ]
                                   │
                                   ▼ (Raw 1Gbps Fiber Line)
                    ┌───────────────────────────────┐
                    │      Intel i3-N300 Core       │
                    │                               │
                    │ • SFP+ Port 1: 1G WAN Bypass  │
                    │   └─ SFP GPON ONT Stick       │
                    │      (1000Base-X / 2.5G Sync) │
                    │      (PLOAM/SLID & WAN 802.1Q)│
                    │                               │
                    │ • SFP+ Port 2: 10G LAN Trunk  │
                    └──────────────┬────────────────┘
                                   │
                                   ▼ (10G SFP+ Direct Attach Copper)
                    ┌───────────────────────────────┐
                    │   MikroTik CRS305-1G-4S+IN    │
                    │   (10G SFP+ Storage/Core Spine)│
                    │   • eth1: Mgmt PoE-In (VLAN 1)│
                    │   • SFP+ 1: 10G Router Uplink │
                    │   • SFP+ 2: High-Speed NAS    │
                    │   • SFP+ 3: Omada EAP773 AP   │
                    │   • SFP+ 4: Omada Switch Trunk│
                    └───────┬───────┬───────┬───────┘
                            │       │       │
       (10G SFP+ DAC) ──────┘       │       └─────► (10G SFP+ DAC Trunk)
                                    ▼               ┌───────────────────────────────┐
       ┌────────────────────────┐  (10G Copper Trx) │  Omada SG2210XMP-M2 Switch    │
       │ AMD Ryzen 5 Node       │   │               │   • 8x 2.5G PoE+ Endpoints    │
       │  • Mellanox ConnectX-3 │   ▼               │     (Pi 5, Cameras, Workstn)  │
       │  • Unraid / ZFS Tier   │  ┌──────────────┐ └───────────────────────────────┘
       └────────────────────────┘  │ Omada EAP773 │
                                   │  Wi-Fi 7 AP  │
                                   └──────────────┘
```

---

---

## 1. Node Topology & Workload Segmentation

### Node 1: Intel i3-N300 Gateway Core [VMID 100 - 299]

- **Compute:** 8C/8T low-power Gracemont architecture, 16GB Crucial DDR5 4800MHz, Samsung PM9A1 NVMe SSD.
- **VMID 100 (OPNsense Router):** 8GB RAM pinned; manages Zenarmor Layer 7 DPI in-memory databases, multi-VLAN policy routing, and outbound GeoIP enforcement.
- **VMID 200 (Identity & App Core LXC):** Unprivileged Linux container hosting Authentik OIDC control plane, Vaultwarden, Securo, and Primary AdGuard Home.

### Node 2: AMD Ryzen 5 Compute & Storage Core [VMID 300 - 499]

- **Compute:** AMD Ryzen 5 3600 (6C/12T), ASRock B550M Pro4, 32GB/64GB DDR4 RAM.
- **Local Hypervisor Tier:** Proxmox VE installed across 2x Intel DC S3500 Enterprise SSDs in a mirrored ZFS pool.
- **Storage Interconnect:** Mellanox ConnectX-3 10G SFP+ linked via DAC Twinax to the MikroTik CRS305 core.
- **VMID 300 (Unraid VM):** Direct PCIe IOMMU passthrough of an **LSI Broadcom SAS 9211-8i HBA Card (IT-Mode)** managing mechanical hard drives alongside a dedicated NVMe SSD cache tier.
- **LXC 310 (DMZ Media):** Unprivileged container with direct NVIDIA GeForce GTX 1660 `/dev/dri` passthrough for hardware-accelerated NVENC transcode pipelines.
- **LXC 320 (NOC Telemetry):** Aggregates metric collectors including Prometheus, Grafana Loki, and visual command panels.

### Node 3: Raspberry Pi 5 Standalone Resiliency Anchor [VMID 500 - 599]

- **Infrastructure Independence:** Powered directly via 2.5G PoE+ HAT inside an aluminum chassis (completely isolated from the Proxmox cluster failure domain).
- **VMID 500 (NUT Master):** Hardwired via USB to a CyberPower CP1500PFCLCD Pure Sine Wave UPS; coordinates automated orderly hypervisor shutdown sequences during power loss events.
- **VMID 520 (Fallback DNS Engine):** Secondary AdGuard Home instance with local `ctrld` proxy routing DoQ and automated failover to Quad9 Oblivious DoH.

---

## 2. High-Speed 10G Storage Fabric

To eliminate network bottlenecks during backup jobs and real-time transcode ingestion, the switching plane is bifurcated:

- **Core Storage Spine (MikroTik CRS305-1G-4S+IN):** Dedicated 10G SFP+ Layer 2 fabric offloading array traffic, local ZFS replication, and high-speed NVMe scratch syncs.
- **PoE Out-of-Band Powering:** The MikroTik is powered via 802.3af/at PoE-in directly from the Omada switch, keeping the 10G spine tied to the primary UPS battery bus.

---

## 3. Storage Strategy & I/O Optimization

- **Atomic Hardlinks:** Unraid array topology leverages a unified parent share (`/mnt/user/data/`), enabling containers to execute instant 0ms hardlink pointer moves between download scratch disks and media libraries without provoking disk thrashing.
- **Two-Tier Disaster Recovery:** Local block-level snapshots route to an isolated Proxmox Backup Server (PBS) datastore, while mission-critical configuration ledgers and application databases are encrypted client-side via Kopia and shipped to Backblaze B2.

---

## 4. Automation & Code Artifacts

### Enterprise Proxmox NUT Shutdown Orchestrator

To solve the risk of disk parity invalidation and database corruption during utility power failures, an automated Python orchestration engine interacts directly with the Proxmox VE 8.x REST API. Triggered by Network UPS Tools (NUT) on the Raspberry Pi 5, it executes ordered guest teardowns (Applications $\rightarrow$ Storage Array $\rightarrow$ Core Gateway) before issuing host-level ACPI poweroffs.

- **GitHub Repository:** [`Brandex9/pve-nut-orchestrator`](https://github.com/Brandex9/pve-nut-orchestrator)
- **Core Capabilities:** Scoped Proxmox RBAC API integration, state polling, asynchronous timeout buffers, and zero-loss storage unmounting.

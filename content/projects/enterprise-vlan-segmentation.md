---
title: "Production Multi-VLAN Network Segmentation & Threat Isolation"
date: 2026-09-30
draft: false
tags: ["OPNsense", "Firewall", "VLANs", "Zenarmor", "Zero-Trust", "Networking"]
summary: "Design and deployment of an enterprise 6-VLAN network topology utilizing virtualized OPNsense, Zenarmor DPI, and default-deny inter-VLAN filtering."
weight: 1
---

## Executive Summary

Transitioned an unmanaged flat consumer network into an enterprise-grade private IP addressing fabric. The deployment enforces strict inter-VLAN default-deny boundaries, layer-7 deep packet inspection (DPI), isolated out-of-band hypervisor management, and an air-gapped security staging sandbox.

---

## 1. Network Topology & Addressing Schema

Routing and packet inspection are handled by an **Intel i3-N300** hardware node running a virtualized **OPNsense Core Router (VMID 100)** over 4x Intel i226-V 2.5GbE interfaces, connected to a **TP-Link Omada SG2210XMP-M2** managed 2.5G PoE+ switch.

| VLAN ID     | Subnet CIDR     | Zone Purpose    | Isolation & Ingress/Egress Rules                                                                                                                                                 |
| :---------- | :-------------- | :-------------- | :------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **VLAN 1**  | `10.1.1.0/24`   | **OOB_MGMT**    | Air-gapped management network for Proxmox VE hypervisor web consoles, Unraid administration dashboards, and switch UI. Inaccessible from WAN or standard subnets.                |
| **VLAN 10** | `10.10.10.0/24` | **APP_CORE**    | Core identity control plane hosting Authentik OIDC forward-auth engine, Vaultwarden password vault, and AdGuard Home DNS.                                                        |
| **VLAN 20** | `10.20.20.0/24` | **TRUSTED_LAN** | Primary workstation access; tightly restricted dynamic scope (`.200-.220`) with static DHCP reservations for authorized physical nodes.                                          |
| **VLAN 30** | `10.30.30.0/24` | **DMZ_MEDIA**   | High-throughput containerized media stack and transcode workloads on the Ryzen compute core.                                                                                     |
| **VLAN 40** | `10.40.40.0/24` | **IOT_SMART**   | Home automation appliances, smart TVs, and IoT microcontrollers. Default-deny rule back into internal RFC1918 subnets; untrusted devices restricted via explicit MAC drop loops. |
| **VLAN 50** | `10.50.50.0/24` | **GUEST_NET**   | Direct-to-WAN guest internet only. DHCP lease times compressed to 2 hours with client isolation enabled.                                                                         |
| **VLAN 99** | `10.99.99.0/24` | **STAGING**     | Sandboxed forensic/test laboratory for packet capturing untrusted containers and new images. Dropped from 100% of lateral internal network routes.                               |

---

## 2. Firewall Policy & Packet Filtering Design

### Inter-VLAN Isolation Matrix

The firewall applies a **Zero-Trust Default-Deny** model across all subnets:

```text
[VLAN 40: IoT]    ──(BLOCKED)──> [VLAN 1, 10, 20, 30]
[VLAN 50: Guest]  ──(BLOCKED)──> [ALL INTERNAL RFC1918]
[VLAN 99: Stage]  ──(BLOCKED)──> [ALL INTERNAL RFC1918 (WAN Outbound Only)]
[VLAN 20: Admin]  ──(ALLOW)────> [VLAN 1, 10, 30, 40] (Stateful return inspection)
```

---

### Step 3: Build the Second Project (Proxmox Compute & Storage Cluster)

Create the virtualization case study at `content/projects/proxmox-virtualized-storage.md`:

```markdown
---
title: "Multi-Node Hypervisor Fabric & Hardware-Accelerated Storage Cluster"
date: 2026-09-30
draft: false
tags: ["Proxmox", "Unraid", "ZFS", "PCIe-Passthrough", "Virtualization"]
summary: "Implementation of a 3-node fault-tolerant virtualization architecture featuring IOMMU hardware passthrough, local ZFS mirror pools, and decoupled power resiliency."
weight: 2
---

## Architecture Overview

A purpose-built hybrid hypervisor architecture running Proxmox VE, combining high-efficiency gateway compute, dedicated high-throughput array storage, and out-of-band power orchestration.

---

## 1. Node Topology & Workload Segmentation

### Node 1: Intel i3-N300 Gateway Core [VMID 100 - 299]

- **Compute:** 8C/8T low-power Gracemont architecture, 16GB Crucial DDR5 4800MHz, Samsung PM9A1 NVMe.
- **VMID 100 (OPNsense Router):** 8GB RAM pinned; manages Zenarmor Layer 7 DPI in-memory databases and multi-VLAN policy routing.
- **VMID 200 (Identity Core LXC):** Unprivileged Linux container hosting Authentik OIDC control plane, Vaultwarden, and Primary AdGuard Home.

### Node 2: AMD Ryzen 5 Compute & Storage Core [VMID 300 - 499]

- **Compute:** AMD Ryzen 5 3600 (6C/12T), ASRock B550M Pro4, 32GB DDR4 3000MHz RAM.
- **Interconnect:** Mellanox ConnectX-3 10G SFP+ linked via DAC Twinax cable directly to core storage distribution.
- **Local Hypervisor Tier:** Proxmox VE operating system installed across 2x Intel DC S3500 Enterprise SSDs in a mirrored ZFS pool.
- **VMID 300 (Unraid VM):** Direct PCIe IOMMU passthrough of an **LSI Broadcom SAS 9211-8i HBA Card (IT-Mode)** managing mechanical hard drives alongside a dedicated Western Digital NVMe SSD cache tier.
- **LXC 310 (DMZ Media):** Unprivileged container with direct NVIDIA GeForce GTX 1660 `/dev/dri` passthrough for hardware-accelerated NVENC transcode pipelines.

### Node 3: Raspberry Pi 5 Resiliency Anchor [VMID 500 - 599]

- **Infrastructure Independence:** Powered via 2.5G PoE+ HAT inside an aluminum chassis (completely isolated from the Proxmox cluster domain).
- **VMID 500 (NUT Master):** USB-tethered to a CyberPower CP1500PFCLCD Pure Sine Wave UPS; coordinates automated orderly hypervisor shutdown sequences during power loss events.
- **VMID 520 (Fallback DNS Engine):** Secondary AdGuard Home synchronizing filters every 15 minutes, with local `ctrld` proxy routing DoQ and automated failover to Quad9 Oblivious DoH.

---

## 2. Storage Strategy & I/O Optimization

- **Atomic Hardlinks:** Storage topology leverages a unified parent share (`/mnt/user/data/`), allowing Docker containers to execute instant 0ms hardlink pointer moves between download scratch disks and media libraries without invoking storage-bus write cycles.
```

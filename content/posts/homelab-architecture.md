---
title: "Enterprise Multi-VLAN Network & Microsegmented Lab Architecture"
date: 2026-09-30
draft: false
tags: ["Networking", "OPNsense", "Proxmox", "VLANs", "Zero-Trust"]
summary: "A production-grade homelab architecture implementing a 6-VLAN network schema, virtualized OPNsense Layer 7 inspection, and hardened storage fabrics."
---

## Architectural Overview

Transitioned from a flat consumer network topology to an enterprise-grade private IP addressing fabric anchored by an Intel i3-N300 OPNsense core router and a TP-Link Omada 2.5G managed switch.

### Key Implementation Highlights

- **6-VLAN Microsegmentation:** Strict inter-VLAN default-deny boundary isolation across Trusted LAN, Out-of-Band Management (OOB_MGMT), Application Core, DMZ Media, IoT Smart, and an isolated Security Staging sandbox.
- **Compute & Storage Fabric:** Bare-metal Proxmox VE hypervisors hosting virtualized Unraid storage (direct PCIe pass-through of LSI SAS 9211-8i in IT Mode) alongside containerized workloads.
- **GitOps Application Orchestration:** Zero plaintext credential storage using Docker Named Secrets (`/run/secrets/`) and Wolfi-hardened container runtimes.
- **Disaster Recovery Strategy:** Multi-tier DR pipeline combining local block-level Proxmox Backup Server (PBS) SSD mirrors with client-side encrypted, deduplicated Kopia snapshots shipped to Backblaze B2.

---
title: "Professional Resume"
layout: "page"
summary: "Brandon Extra — Network Security Specialist & Aspiring NOC Analyst"
---

# Brandon Extra

**Avenel, NJ** | [brandex@brandonextra.com](mailto:brandex@brandonextra.com) | [GitHub](https://github.com/Brandex9) | [LinkedIn](https://linkedin.com/in/extrab9)

---

## Professional Summary

Certified IT & Network Security Specialist blending 9+ years of operational leadership at Amazon, with advanced technical training in cloud architecture, network infrastructure, and virtualization. Transitioning into a Help Desk / NOC Analyst role to combine a strong background in real-time incident routing with a deep technical curiosity for cybersecurity and user support. Highly proficient in configuring enterprise firewall environments (OPNsense), managing hardware hypervisors (Proxmox), deploying hybrid cloud perimeters (VPS), and performing root-cause analysis. Possesses a strong ability to read, analyze, and parse multiple programming languages to diagnose backend errors and script system automations.

---

## Technical Proficiencies

- **Networking & Firewalls:** OPNsense, Zenarmor (DPI), Multi-VLAN Microsegmentation, WireGuard Site-to-Site, DNS-over-QUIC (DoQ), mDNS routing, 802.1Q Trunks, 10G SFP+ Interconnects.
- **Systems & Hypervisors:** Proxmox VE (Clustered & Standalone), Debian/Ubuntu Server, Alpine Linux, Unraid OS, IOMMU/PCIe Passthrough (HBAs, NVENC GPUs).
- **Containerization & Orchestration:** Docker, Rootless Containers, Docker Named Secrets (`/run/secrets/`), Dockhand GitOps, Hawser Agents.
- **Security & Identity:** Authentik (OIDC/SAML/Forward-Auth), Vaultwarden, CrowdSec, Fail2ban, Least-Privilege Role-Based Access Control (RBAC).
- **Storage & Resiliency:** ZFS (Mirroring, Local Datastores), IT-Mode LSI HBAs, Kopia Deduplicated Backups, Proxmox Backup Server (PBS), Backblaze B2, Network UPS Tools (NUT).
- **Monitoring & SIEM:** Prometheus, Grafana, Loki Log Aggregation, ICMP/SNMP Health Polling.

---

## Hands-On Infrastructure Projects

### Enterprise Homelab Virtualization & 6-VLAN Fabric | _Avenel, NJ_

- **Architecture:** Engineered a production-grade 6-VLAN network schema separating Trusted LAN, Air-Gapped OOB Management, Application Core, DMZ Media, IoT, and Security Sandboxing.
- **Firewall Engineering:** Deployed a virtualized OPNsense core router running Zenarmor Layer 7 inspection with strict inter-VLAN default-deny boundary rules and IPv6 leak containment.
- **Storage Cluster:** Implemented direct PCIe IOMMU passthrough of an LSI SAS 9211-8i HBA to an Unraid storage VM, backed by local ZFS mirror pools on enterprise Intel DC SSDs.
- **Automated DR:** Orchestrated a two-tier disaster recovery architecture combining local block-level PBS snapshots with offsite AES-256 client-side encrypted Kopia snapshots synced to Backblaze B2.

---

## Professional Experience

Amazon | Operations & Incident Logistics Leader 2017 – Present

- Orchestrate real-time incident routing and queue prioritization for high-volume delivery operations, maintaining 99.8% SLA compliance under tight operational thresholds.
- Conduct root-cause analysis (RCA) on process bottlenecks and operational telemetry failures, implementing data-driven remediation workflows across cross-functional teams.
- Lead and mentor large, fast-paced teams, translating complex metrics into actionable dispatching queues and managing multi-channel escalation paths during high-severity system exceptions.

## Certifications & Education

- **Security & Networking Certifications:** _CompTIA A+ | Google Cybersecurity Professional Certificate | NJIT Software Development Professional Certificate_
- **Active Pursuit:** _CompTIA Security+ | Cisco Certified Network Associate (CCNA) | AWS Certified Solutions Architect – Associate_
- **Continuous Education:** Hands-on Packet Analysis, Linux Hardening, and Intrusion Detection Labs.

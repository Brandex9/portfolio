---
title: "Decoupled DNS Architecture: Sub-Millisecond DoQ Privacy Plane"
date: 2026-10-09
draft: false
tags:
  [
    "Networking",
    "DNS",
    "AdGuard-Home",
    "Control-D",
    "OPNsense",
    "Zero-Trust",
    "Infrastructure",
  ]
summary: "Architecting a high-performance DNS-over-QUIC (DoQ) pipeline that strips residential IP footprints and eliminates recursive multi-binary context overhead."
weight: 3
---

## Executive Summary

Chained local recursive resolvers (e.g., Pi-hole looped into Unbound instances) introduce processing redundancies, application-layer context-switching latency, and flash storage write saturation.

This deployment implements a decoupled, high-frequency **DNS-over-QUIC (DoQ)** architecture. Local regex filtering and master RAM caching are consolidated within an AdGuard Home container, while cache misses route directly into a native Control D daemon (`ctrld`) on the core gateway. Upstream queries are cryptographically sealed with TLS 1.3 over UDP and routed across a kernel-level WireGuard tunnel to a stateless OVHcloud edge VPS, ensuring public Anycast resolvers never correlate residential IP identities with user query streams.

---

## 1. Streamlined Wirespeed Transit Pipeline

Rather than passing DNS queries through a convoluted multi-hop resolver chain, this architecture flattens the local resolving fabric into a non-blocking **3-step wirespeed data plane**:

```text
 [ Local Client Workstation ]
                │
                ▼ (Port 53 / Local Network Segment)
┌────────────────────────────────────────────────────────┐
│ 1. AdGuard Home Container (VLAN 10: APP_CORE)          │
│    • Master Ad/Malware Regex Filtering Plane           │
│    • MASTER CACHE: Enforces 1-Hour Prefetch Tables     │
└──────────────┬─────────────────────────────────────────┘
               │ (Cache Misses: Forwarded down VLAN trunk)
               ▼
┌────────────────────────────────────────────────────────┐
│ 2. Control D Proxy Daemon (ctrld running on OPNsense)  │
│    • Ingests raw UDP stream on OPNsense Port 5354      │
│    • Packages query directly into DNS-over-QUIC (DoQ)  │
└──────────────┬─────────────────────────────────────────┘
               │
               ▼ (OPNsense Policy Route forces packet into Tunnel)
┌────────────────────────────────────────────────────────┐
│ 3. Kernel-Level WireGuard Tunnel Interface             │
│    • Hardened Link MTU Clamped to 1420 bytes           │
└──────────────┬─────────────────────────────────────────┘
               │
               ▼ (Traverses Public WAN completely masked)
┌────────────────────────────────────────────────────────┐
│ 4. OVHcloud VPS Edge ➔ Outbound DoQ to Control D      │
│    • Decapsulate WireGuard ➔ Forwards request(Port853)│
│    • Identity fully decoupled via cloud IP footprint   │
└────────────────────────────────────────────────────────┘
```

---

## 2. Infrastructure Interface & Addressing Profile

To guarantee consistent log parsing and automated configuration mapping across the hypervisor fabric, the data plane is harmonized under a deterministic **`10.VLAN.VLAN.X`** schema:

| Infrastructure Component | Network Segment         | Static IP Mapping | Primary Operational Mandate                                             |
| :----------------------- | :---------------------- | :---------------- | :---------------------------------------------------------------------- |
| **AdGuard Home Node**    | `VLAN 10` (APP_CORE)    | `10.10.10.15`     | Local LAN Port 53 listener; handles regex tables and RAM cache lookups. |
| **OPNsense Gateway**     | Core Firewall Interface | `10.10.10.1`      | Gateway router; accepts cache misses from AdGuard on port 5354.         |
| **Control D Daemon**     | Local Service Engine    | `127.0.0.1:5354`  | Native `ctrld` binary converting UDP frames into secure DoQ streams.    |
| **WireGuard Transit**    | Point-to-Point Tunnel   | `10.99.99.2/30`   | Kernel-level encrypted pipe bound to stateless Canada East cloud VPS.   |

---

## 3. Cryptographic Protocol Engineering: The DoQ Advantage

Traditional encrypted transport protocols like DNS-over-TLS (DoT) and DNS-over-HTTPS (DoH) run atop TCP, introducing multi-step handshake latency and **Head-of-Line (HoL) Blocking**—where a single dropped packet stalls all subsequent queries in the queue.

This architecture standardizes all upstream transit on native **DNS-over-QUIC (DoQ)** on Port 853, pointing directly to **Control D’s Anycast infrastructure**:

- **Symmetric 1-RTT Handshakes:** Operating natively over UDP, QUIC merges TLS 1.3 cryptographic negotiation and transport initialization into a single round trip, dropping cold lookup times to ~12ms.
- **Connection Migration Resiliency:** Roaming endpoints traversing through the **GL.iNet Slate 7 Pro travel router** maintain active DNS sessions across hotel Wi-Fi and cellular failovers without renegotiation loops.
- **Tunnel MTU Hardening:** To eliminate packet drops and IP fragmentation inside the WireGuard tunnel, the interface MTU is clamped to **1420 bytes**, ensuring large DNS responses fit cleanly within the transport envelope.

---

## 4. Hardware Offloading & Storage Endurance

High-frequency query logging across dozens of home IoT and media endpoints causes severe write amplification on boot storage. To protect the **Pentium Gold 8505 appliance SSDs**, a multi-tiered quarantine policy is enforced:

- **Volatile Caching:** AdGuard Home query logging is restricted to a **24-hour circular buffer** retained purely within volatile system RAM.
- **Telemetry Decoupling:** Long-term analytics are extracted via Prometheus and Loki collectors, compressed into time-series blocks, and stored on an isolated secondary storage pool.
- **Encrypted Snapshot Shipping:** Transactional metric blocks are cataloged nightly by the **Kopia backup engine**, encrypted via client-side **AES-256-GCM**, and pushed to Backblaze B2 object storage.

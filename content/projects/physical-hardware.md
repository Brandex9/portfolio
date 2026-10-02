---
title: "Physical Layer Blueprint: Non-Blocking 10G SFP+ Core Fabric"
date: 2026-10-02
draft: false
tags: ["Networking", "OPNsense", "MikroTik", "Omada", "Hardware", "10GbE"]
summary: "Engineering a multi-gigabit hardware switching matrix pairing an Intel i3-N300 OPNsense core router with a 10G SFP+ MikroTik aggregation spine and an Omada PoE+ edge plane."
weight: 5
---

## Engineering Overview

A hardened software security perimeter is only as resilient as the underlying physical topology. This project details the design and deployment of a non-blocking, multi-gigabit infrastructure fabric designed to handle intra-VLAN routing saturation and Wi-Fi 7 backhaul throughput without introducing Layer 2 bottlenecks.

```text
                        [ WAN / 10G Fiber ONT ]
                                │
                                ▼ (10G SFP+ Fiber/Copper)
                ┌───────────────────────────────┐
                │    Intel i3-N300 OPNsense     │
                │     • SFP+ Port 1: WAN        │
                │     • SFP+ Port 2: LAN Core   │
                └───────────────┬───────────────┘
                                │
                                ▼ (10G SFP+ Direct Attach Copper)
                ┌───────────────────────────────┐
                │    MikroTik CRS305 Switch     │
                │     • Port 1: Uplink from Rtr │
                │     • Port 2: Dnlink to Omada │
                │     • Port 3: Dnlink to EAP773│
                │     • Port 4: High-Speed NAS  │
                └───────────────┬───────┬───────┘
                                │       │
      (10G SFP+ DAC / Optics)   │       │ (10G SFP+ to 10Gbase-T Transceiver)
                                ▼       ▼
┌─────────────────────────────────┐   ┌─────────────────────────────────┐
│  Omada SG2210XMP-M2 Edge Switch │   │   Omada EAP773 Wi-Fi 7 AP       │
│   • SFP+ Port 1: Core Uplink    │   │    • 10G RJ45 Port              │
│   • 8x 2.5G PoE+ Endpoints      │   │    • Out-of-Band 12V DC Power   │
└─────────────────────────────────┘   └─────────────────────────────────┘
```

---

## 1. Hardware Inventory & Interface Allocations

To minimize user-space overhead and context-switching latencies on the firewall, high-bandwidth storage and wireless backhaul workloads are offloaded directly to a dedicated Layer 2 switching spine:

- **Core Routing Engine (Intel i3-N300 Appliance):** Equipped with 2× 10G SFP+ and 3× 2.5GbE interfaces. Port 1 handles raw 10G WAN ingestion. Port 2 executes a 10G symmetric LAN backbone trunk.
- **Aggregation Spine (MikroTik CRS305):** Deployed natively in **SwOS (SwitchOS)** mode to completely bypass Layer 3 processing overhead, enabling wirespeed Layer 2 non-blocking matrix forwarding across 4× 10G SFP+ cages.
- **Access Plane & Power Distribution (Omada SG2210XMP-M2):** Acts as the Multi-Gig access layer, feeding 8× 2.5GbE PoE+ copper links to endpoints while leveraging a 10G SFP+ uplink back to the core aggregation spine.
- **Wireless Backhaul (Omada EAP773 Wi-Fi 7):** Utilizes an onboard 10Gbase-T interface to sustain peak multi-band wireless throughput without clipping at the copper link layer.

---

## 2. Interconnect & Media Engineering

### Wirespeed 10G SFP+ Backbone

The connection mapping between the OPNsense firewall, the MikroTik spine, and the Omada edge switch utilizes passive **10G Direct Attach Copper (DAC) twinaxial cables**. This methodology keeps link latency down to sub-microsecond thresholds and strips out the active power/thermal overhead introduced by standard 10Gbase-T copper transceivers.

### High-Speed Wi-Fi 7 Transceiver Adaptations

Bridging the MikroTik SFP+ spine to the native 10Gbase-T RJ45 port on the Omada EAP773 access point requires converting optical/differential signals into baseband copper. This is accomplished using a multi-rate **10G SFP+ to RJ45 Copper Transceiver Module** configured to sync at full 10Gbps line rate over Cat6A shielded twisted-pair cabling.

---

## 3. Power Isolation & VLAN Frame Management

### Out-of-Band PoE++ Remediation

The Omada EAP773 requires a strict **802.3bt PoE++** envelope (or dedicated 12V DC input) to drive its high-density Tri-Band radio arrays. Because the Omada edge switch is natively capped at an 802.3at PoE+ budget (30W max per port), the access point is intentionally decoupled from switch power and driven via an independent **out-of-band 12V DC adapter** to prevent thermal throttling or frame degradation under peak utilization.

### 802.1Q Frame Gating Strategy

- **Trunk Profiles:** Inter-switch links are provisioned as strict 802.1Q tagged boundaries. Management frames run untagged on Native VLAN 1, while user traffic profiles are tightly encapsulated within distinct tags (VLAN 10 for App Core, VLAN 20 for Trusted LAN, VLAN 30 for Staging/IoT).
- **SSID-to-VLAN Mapping:** The EAP773 ingests the tag profile directly from the MikroTik aggregation link. Internal broadcast domains map directly to dedicated virtual radio interfaces, isolating insecure smart home peripherals at the physical air interface before packets ever hit the core switching fabric.

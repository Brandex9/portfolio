---
title: "Intent-Based Zero Trust: Tri-Tier Remote Access & Ingress Microsegmentation"
date: 2026-10-03
draft: false
tags:
  [
    "Zero-Trust",
    "ZTNA",
    "Twingate",
    "Tailscale",
    "Authentik",
    "WireGuard",
    "Networking",
  ]
summary: "Partitioning enterprise remote access into specialized operational scopes using Pangolin IP-gating, Twingate least-privilege ZTNA, and a Tailscale travel mesh."
weight: 3
---

## Executive Summary

Deploying a single monolithic VPN introduces lateral movement risks and breaks non-technical user workflows. This deployment implements **Intent-Based Access Control (IBAC)**, partitioning remote access into three isolated planes: application-level ZTNA for administration, automated hardware-level mesh routing for mobile travel, and dynamic IP-whitelisted edge proxying for consumer client hardware.

---

## 1. Architectural Allocation Matrix

| Access Vector                    | Technology Tier          | Target Scope                            | Security Policy & Ingress Mechanics                                                                                                                                       |
| :------------------------------- | :----------------------- | :-------------------------------------- | :------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| **Consumer Application Ingress** | **Pangolin + Authentik** | Media & Streaming Workloads (`VLAN 30`) | Ingests native smart TV/client app requests without VPN clients. Authentik fires webhooks upon WebAuthn validation to dynamically whitelist consumer WAN IPs on Pangolin. |
| **Administrative Control Plane** | **Twingate (ZTNA)**      | Core Management (`VLAN 10` & `VLAN 1`)  | Enforces micro-segmented, least-privilege resource mapping. Grants point-to-point socket access (e.g., `10.1.1.2:8006`) without exposing Layer 3 subnets.                 |
| **Infrastructure Mesh Backhaul** | **Tailscale**            | Physical Node Interconnect              | Encrypted WireGuard overlay bridging remote hardware (GL.iNet Slate 7 Pro) to OPNsense via Subnet Routing and Split-Horizon DNS.                                          |

---

## 2. Ingress & Routing Topology

                                     [ THE PUBLIC WAN INTERNET ]
                                                    │
                    ┌───────────────────────────────┼──────────────────────────────┐
                    ▼ (Public Web / Media Ingress)  ▼ (ZTNA Admin Traffic)         ▼ (Mesh Satellite Bridge)
         ┌──────────────────────┐        ┌──────────────────────┐       ┌──────────────────────┐
         │   Cloudflare Edge    │        │   Twingate Relays    │       │ Tailscale DERP Fleet │
         │   • DDoS/WAF Gating  │        │   • Identity Brokered│       │ • Encrypted P2P Mesh │
         └──────────┬───────────┘        └──────────┬───────────┘       └──────────┬───────────┘
                    │ (Port 443)                    │                              │
                    ▼                               │ (Direct Outbound Egress)     │ (Direct Outbound Egress)
         ┌──────────────────────┐                   │                              │
         │  OVHcloud VPS Edge   │                   │                              │
         │  • Caddy L4/L7 SNI   │                   │                              │
         │  • GeoIP / CrowdSec  │                   │                              │
         └──────────┬───────────┘                   │                              │
                    │                               │                              │
                    │ (Outbound WireGuard /30)      │                              │
                    ▼                               ▼                              ▼
     ┌─────────────────────────────── HOME FIREWALL BOUNDARY ────────────────────────────────────────┐
     │                                                                                               │
     │   ┌───────────────────────────────────────────────────────────────────────────────────────┐   │
     │   │ VMID 100: OPNsense Core Gateway (Intel i3-N300 Node)                                  │   │
     │   │  • Static Inbound WAN Ports: 0 OPEN (All Ingress Outbound-Initiated)                  │   │
     │   │  • Tailscale Subnet Router (Mesh Bridge to Travel Slate 7 Pro)                        │   │
     │   │  • Zenarmor Layer 7 DPI (Deep Packet Inspection on Inter-VLAN Traffic)                │   │
     │   └──────────────────────────────────────────┬────────────────────────────────────────────┘   │
     └──────────────────────────────────────────────┼────────────────────────────────────────────────┘
                                                    │ (802.1Q Trunk / 10G MikroTik CRS305 Spine)
                                                    ▼
     ┌───────────────────────────────── COMPUTATION & CLUSTER FABRIC ────────────────────────────────┐
     │                                                                                               │
     │   [ NODE 1: Intel i3-N300 Core ]                                                              │
     │   ┌─────────────────────────────── VMID 200: App Core Container (VLAN 10) ────────────────┐   │
     │   │                                                                                       │   │
     │   │  ┌────────────────────────┐                   ┌────────────────────────┐              │   │
     │   │  │ Authentik IdP Core     │                   │ Twingate Connector     │              │   │
     │   │  │ • 10.10.10.10          │                   │ • 10.10.10.20          │              │   │
     │   │  │ • WebAuthn / OIDC      │                   │ • Scoped Resource Broker              │   │
     │   │  └───────────┬────────────┘                   └────────────────────────┘              │   │
     │   └──────────────│────────────────────────────────────────────────────────────────────────┘   │
     │                  │                                                                            │
     │                  │ (Cross-VLAN Webhook Auth / Dynamic IP Allow)                               │
     │                  ▼                                                                            │
     │   [ NODE 2: AMD Ryzen 5 Core ]                                                                │
     │   ┌──────────────┴──────────────── LXC 310: DMZ Media Cluster (VLAN 30) ──────────────────┐   │
     │   │                                                                                       │   │
     │   │  ┌────────────────────────┐                   ┌────────────────────────┐              │   │
     │   │  │ Pangolin Ingress Proxy │                   │ Jellyfin Media Server  │              │   │
     │   │  │ • 10.30.30.30          │──────────────────►│ • 10.30.30.40          │              │   │
     │   │  │ • Dynamic IP Whitelist │  (Local Stream)   │ • Hardware NVENC Trans │              │   │
     │   │  └────────────────────────┘                   └────────────────────────┘              │   │
     │   └───────────────────────────────────────────────────────────────────────────────────────┘   │
     └───────────────────────────────────────────────────────────────────────────────────────────────┘

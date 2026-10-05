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

```text
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
```

---

## 3. Engineering Implementation Details

### Tier 1: Pangolin & Authentik Dynamic Webhook Gating

Consumer smart TVs and streaming appliances cannot process interactive HTTP 302 redirects triggered by authentication forward proxies.

- Pangolin runs isolated inside **DMZ_MEDIA (VLAN 30)** on the AMD Ryzen 5 compute node.
- When family members or mobile clients complete FIDO2/WebAuthn verification on their personal browser, an event-driven webhook executes an API call against Pangolin to dynamically whitelist the originating residential WAN IP for a 24-hour lease window.

### Tier 2: Twingate Least-Privilege Administrative Access

- The Twingate Connector is deployed as an unprivileged container in **APP_CORE (VLAN 10)**.
- Point-to-point software-defined perimeters replace traditional Layer 3 VPN connections. Administrators connect exclusively to designated host:port definitions (e.g., `10.1.1.2:8006` for the Proxmox UI), preventing lateral discovery or network-wide port scans.

### Tier 3: Tailscale Mesh with GL.iNet Hardware Integration

- OPNsense on the Intel N300 runs Tailscale as a kernel-accelerated Subnet Router.
- A portable **GL.iNet Slate 7 Pro** travel router acts as a mobile satellite node configured on `10.150.1.0/24`. Client devices connected to the mobile travel Wi-Fi automatically resolve internal DNS records (`*brandonextra.com`) through the secure mesh backhaul without per-device client software.

### Tier 4: Internal Metasearch Microsegmentation (SearXNG)

Publicly exposing self-hosted metasearch instances (e.g., SearXNG) creates immediate operational issues: search engine scrapers and botnets saturate upstream quotas, prompting search providers (Google, Bing, Brave) to flag the public IP with persistent reCAPTCHAs and rate limits.

To eliminate public exposure while retaining native search engine integration across managed devices, SearXNG is isolated under an **Intent-Based Access Control (IBAC)** model:

```text
  [ Admin Workstation / Mobile Browser ]
                     │
                     ▼ (Encrypted ZTNA Tunnel Request)
           [ Twingate Client Agent ]
                     │
                     ▼ (Point-to-Point Socket Authorization)
           [ Twingate Connector ] ──► [ SearXNG Container ]
             (App Core: VLAN 10)         (App Core: 10.10.10.35:8080)
                                                 │
                                                 ▼ (Policy Route: Residential WAN Egress)
                                         [ OPNsense Core Gateway ]
```

---

## 4. Production Automation & Code Artifacts

### Authentik-to-Pangolin Dynamic IP Synchronizer

To bridge headless client devices with zero-trust forward authentication, a custom event-driven Python microservice integrates Authentik's notification webhook bus directly with Pangolin's REST API. When an authenticated user completes WebAuthn verification, the engine extracts the validated client WAN IP (stripping multi-hop proxy headers) and provisions a temporary whitelist lease on the media ingress proxy.

- **GitHub Repository:** [`Brandex9/authentik-pangolin-sync`](https://github.com/Brandex9/authentik-pangolin-sync)
- **Core Capabilities:** Upstream reverse-proxy header sanitization (`CF-Connecting-IP`, `X-Forwarded-For`), scoped API bearer token authentication, dynamic TTL lease management, and integrated health probes.

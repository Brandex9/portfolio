---
title: "Hybrid Cloud Perimeter: VPS Reverse-Proxy Shield & Zero-Trust ZTNA"
date: 2026-09-30
draft: false
tags: ["Zero-Trust", "WireGuard", "Cloudflare", "Caddy", "ZTNA", "Security"]
summary: "Architecting an obfuscated edge perimeter using a stateless OVHcloud VPS, Caddy Layer 4 filtering, MaxMind GeoIP gating, and a kernel-level WireGuard transit pipe."
weight: 4
---

## Executive Summary

Exposing residential IP addresses or opening inbound firewall ports introduces severe perimeter risks. This project details the design and deployment of an obfuscated edge architecture that enables secure public accessibility and zero-trust remote administration while keeping **100% of residential ingress ports closed**.

                                [ Public Ingress / WAN ]
                                             │
                                             ▼
                          ┌──────────────────────────────────────┐
                          │           Cloudflare Edge            │
                          │  • Edge DDoS & WAF                   │
                          │  • Injects 'CF-Connecting-IP'        │
                          │  • DNS-01 ACME Challenge API         │
                          └──────────────────┬───────────────────┘
                                             │ (HTTPS / Port 443)
                                             ▼
                          ┌──────────────────────────────────────┐
                          │       OVHcloud VPS (Canada East)     │
                          │                                      │
                          │  1. Caddy Layer 4 Engine             │
                          │     └─ TCP 443 raw stream ingestion  │
                          │                                      │
                          │  2. Caddy Layer 7 HTTP Router        │
                          │     ├─ trusted_proxies cloudflare    │
                          │     ├─ MaxMind GeoIP (US/CA only)    │
                          │     └─ CrowdSec Edge Remediation     │
                          │                                      │
                          │  3. Pre-Auth Ingress Check           │
                          │     └─ Forward-Auth / OIDC Challenge │
                          │        (Validated against Authentik) │
                          └──────────────────┬───────────────────┘
                                             │
                      ▲                      │
                      |                      |
      (Outbound Dial) │                      │ WireGuard /30 Transit Tunnel
    (Zero Home Ports) |                      │ │ (Keepalive = 25s)
                      |                      │ ▼
        ┌─────────────┴───────────────────────────────────────────────────────┐
        │ Home Perimeter: Intel i3-N300 Core                                  │
        │ (Bare-Metal Proxmox VE + Beszel Agent)                              │
        │                                                                     │
        │ ┌───────────────────────────────────────────────────────────────┐   │
        │ │ VMID 100: OPNsense Core Gateway                               │   │
        │ │                                                               │   │
        │ │ 🛡️ Threat Inspection & Outbound Controls:                    │    │
        │ │ • Zenarmor (Layer 7 DPI on Inter-VLAN traffic)                │   │
        │ │ • CrowdSec LAPI Bouncer (Firewall-level drop tables)          │   │
        │ │ • MaxMind GeoIP (Outbound Egress Block for IoT/Staging)       │   │
        │ │                                                               │   │
        │ │ 🌐 Internal Routing & Edge Ingress:                           │   │
        │ │ • os-caddy (Internal TLS termination & local VLAN routing)    │   │
        │ │ • Tailscale Subnet Router (Admin mesh overlay fallback)       │   │
        │ │ • Unbound DNS (Internal split-horizon & DHCP resolver)        │   │
        │ │ • os-ddclient (Dynamic DNS sync) & git-backup (IaC configs)   │   │
        │ └───────────────────────────────┬───────────────────────────────┘   │
        │ │ Inter-VLAN Routing Policy                                         │
        │ ┌───────────────────────────────▼───────────────────────────────┐   │
        │ │ VMID 200: Central Identity & App Core (VLAN 10)               │   │
        │ │ (Unprivileged LXC Container running Docker Compose)           │   │
        │ │                                                               │   │
        │ │ 🔐 Identity & Access Management:                              │   │
        │ │ • Authentik Core (Central OIDC / WebAuthn IdP)                │   │
        │ │ • Vaultwarden (Zero-knowledge password vault)                 │   │
        │ │                                                               │   │
        │ │ 🛡️ DNS & Privacy Plane:                                       │   │
        │ │ • AdGuard Home Primary (Ad-blocking, DoQ to Control D)        │   │
        │ │                                                               │   │
        │ │ 📊 Applications, Financials & Portals:                        │   │
        │ │ • Securo (Self-hosted personal finance engine)                │   │
        │ │ • Homepage (Centralized infrastructure launchpad)             │   │
        │ │                                                               │   │
        │ │ 📈 Telemetry, Monitoring & Backups:                           │   │
        │ │ • Uptime Kuma (Internal HTTP/TCP service health checks)       │   │
        │ │ • Beszel Container Agent (System metrics exporter)            │   │
        │ │ • Kopia Client (Automated daily snapshots to Backblaze B2)    │   │
        │ └───────────────────────────────────────────────────────────────┘   │
        └─────────────────────────────────────────────────────────────────────┘

---

## 1. Cloud-to-Edge Transit Pipeline

### Stateless Cloud Sentry (OVHcloud VPS)

- **Compute:** 2 vCPU, 4GB RAM, running Debian 13 (Trixie) in Beauharnois, QC.
- **Edge Ingress Hardening:** Caddy Server compiled with custom Layer 4 extensions. Ingests raw TCP/UDP streams at the transport layer, while utilizing Caddy's Layer 7 HTTP router to evaluate proxy headers (CF-Connecting-IP) against a local MaxMind GeoIP database to drop unauthorized regional requests at the edge.
- **Administrative Isolation:** Public SSH daemon disabled on public WAN interfaces. Edge instance management and logging execute exclusively across the secure internal WireGuard transport layer.

### Outbound-Initiated `/30` WireGuard Transit Tunnel

- Rather than the VPS initiating connections into the home network, the home Intel N300 OPNsense gateway opens an **outbound-initiated kernel-level WireGuard session** to the VPS.
- Configured as a tightly scoped `/30` point-to-point transit link. In OPNsense, firewall rules explicitly drop any connection attempts originating from the VPS subnet that target hypervisor management planes (VLAN 2) or secure client workstations (VLAN 1).

---

## 2. Zero-Trust Remote Access (ZTNA & Mesh Overlay)

Dual-layered remote connectivity is enforced based on risk posture and device identity:

1. **Hypervisor Data-Plane Orchestration:** Tailscale mesh overlay with Split DNS routing internal domains (`*.extra-infra.net`) directly across the WireGuard transport layer.
2. **Application-Level Micro-Segmentation:** Twingate ZTNA gateways deployed within container runtimes. Access policies enforce least-privilege resource assignments—remote devices access single IP:Port endpoints without granting network-wide Layer 3 subnet exposure.
3. **Hardware Key Authentication:** Centralized Authentik Identity Provider (IdP) enforcing FIDO2/WebAuthn hardware security keys and OIDC claims before authorizing remote tunnels.

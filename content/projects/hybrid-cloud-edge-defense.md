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

            [Public Ingress]
                    │
                    ▼
    [Cloudflare Edge: WAF & DNS-01 ACME]
                    │
                    ▼
    [OVHcloud VPS (Canada East)]
                    ├── Caddy Layer 4 Ingress (MaxMind GeoIP Drop: Non-US/CA)
                    ├── Authelia Pre-Authentication Challenge
                    └── CrowdSec Edge Remediation Bouncer
                    │
                    ▼ (Outbound-Initiated Kernel WireGuard Tunnel /30)
    [Home Perimeter: Intel N300 OPNsense Core]
                    │
            (Strict Policy Route)
                    ▼
    [Internal Workloads: VLAN 10/20]

---

## 1. Cloud-to-Edge Transit Pipeline

### Stateless Cloud Sentry (OVHcloud VPS)

- **Compute:** 2 vCPU, 4GB RAM, running Debian 13 (Trixie) in Beauharnois, QC.
- **Edge Ingress Hardening:** Caddy Server compiled with custom Layer 4 extensions. Ingests raw TCP/UDP streams and matches client connection origins against a local MaxMind GeoIP database to drop non-US/CA packets at the transport layer.
- **Administrative Isolation:** Public SSH daemon disabled on public interfaces. Edge management executes exclusively across an internal WireGuard tunnel.

### Outbound-Initiated `/30` WireGuard Transit Tunnel

- Rather than the VPS initiating connections into the home network, the home Intel N300 OPNsense gateway opens an **outbound-initiated kernel-level WireGuard session** to the VPS.
- Configured as a tightly scoped `/30` point-to-point transit link. In OPNsense, firewall rules explicitly drop any connection attempts originating from the VPS subnet that target hypervisor management planes (VLAN 2) or secure client workstations (VLAN 1).

---

## 2. Zero-Trust Remote Access (ZTNA & Mesh Overlay)

Dual-layered remote connectivity is enforced based on risk posture and device identity:

1. **Hypervisor Data-Plane Orchestration:** Tailscale mesh overlay with Split DNS routing internal domains (`*.extra-infra.net`) directly across the WireGuard transport layer.
2. **Application-Level Micro-Segmentation:** Twingate ZTNA gateways deployed within container runtimes. Access policies enforce least-privilege resource assignments—remote devices access single IP:Port endpoints without granting network-wide Layer 3 subnet exposure.
3. **Hardware Key Authentication:** Centralized Authentik Identity Provider (IdP) enforcing FIDO2/WebAuthn hardware security keys and OIDC claims before authorizing remote tunnels.

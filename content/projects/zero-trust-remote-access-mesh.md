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
summary: "Partitioning enterprise remote access into specialized operational scopes using Caddy edge routing, Authentik centralized IAM, Twingate least-privilege ZTNA, and a Tailscale travel mesh."
weight: 4
---

## Executive Summary

Deploying a single monolithic VPN introduces lateral movement risks and breaks non-technical user workflows. This architecture implements **Intent-Based Access Control (IBAC)**, partitioning remote connectivity into three isolated operational planes: a hardened edge reverse proxy with federated IAM for consumer and media delivery, application-level ZTNA for administrative control, and an automated hardware mesh for mobile site-to-site travel.

---

## 1. Architectural Allocation Matrix

| Access Vector                    | Technology Tier                   | Target Scope                             | Security Policy & Ingress Mechanics                                                                                                                                                                                                                                                                                               |
| :------------------------------- | :-------------------------------- | :--------------------------------------- | :-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Consumer Application Ingress** | **Caddy + Authentik (OIDC/LDAP)** | Web Apps (`VLAN 10`) & Media (`VLAN 30`) | OVHcloud VPS edge terminates public TLS. Web interfaces enforce Authentik OIDC/Forward-Auth challenges. Native streaming clients (Neptune, Apple TV, Android TV) bypass HTML SSO redirects to query Jellyfin APIs directly, authenticating transparently against Authentik's LDAP Outpost with server-enforced parental controls. |
| **Administrative Control Plane** | **Twingate (ZTNA)**               | Core Management (`VLAN 10` & `VLAN 1`)   | Enforces micro-segmented, least-privilege resource mapping. Grants point-to-point socket access (e.g., `10.10.10.2:8006` Proxmox UI) without exposing underlying Layer 3 subnets or broadcasting route tables.                                                                                                                    |
| **Infrastructure Mesh Backhaul** | **Tailscale**                     | Physical Node Interconnect               | Encrypted WireGuard overlay bridging remote hardware (GL.iNet Slate 7 Pro) to OPNsense via kernel Subnet Routing and Split-Horizon DNS.                                                                                                                                                                                           |

---

## 2. Ingress & Routing Topology

```text
                                     [ THE PUBLIC WAN INTERNET ]
                                                  │
                    ┌─────────────────────────────┼────────────────────────────┐
                    ▼ (Public Web / Media Ingress)▼ (ZTNA Admin Traffic)       ▼ (Mesh Satellite Bridge)
         ┌──────────────────────┐      ┌──────────────────────┐     ┌──────────────────────┐
         │   Cloudflare Edge    │      │   Twingate Relays    │     │ Tailscale DERP Fleet │
         │   • DDoS/WAF Gating  │      │   • Identity Brokered│     │ • Encrypted P2P Mesh │
         └──────────┬───────────┘      └──────────┬───────────┘     └──────────┬───────────┘
                    │ (Port 443)                  │                            │
                    ▼                             │ (Direct Outbound Egress)   │ (Direct Outbound Egress)
         ┌──────────────────────┐                 │                            │
         │  OVHcloud VPS Edge   │                 │                            │
         │  • Caddy L7 Ingress  │                 │                            │
         │  • GeoIP / CrowdSec  │                 │                            │
         └──────────┬───────────┘                 │                            │
                    │                             │                            │
                    │ (Outbound WireGuard /30)    │                            │
                    ▼                             ▼                            ▼
     ┌─────────────────────────────── HOME FIREWALL BOUNDARY ────────────────────────────────────────┐
     │                                                                                               │
     │   ┌───────────────────────────────────────────────────────────────────────────────────────┐   │
     │   │ VMID 100: OPNsense Core Gateway (Intel 8505 Node)                                     │   │
     │   │  • Static Inbound WAN Ports: 0 OPEN (All Ingress Outbound-Initiated)                  │   │
     │   │  • Tailscale Subnet Router (Mesh Bridge to Travel Slate 7 Pro)                        │   │
     │   │  • Zenarmor Layer 7 DPI (Deep Packet Inspection on Inter-VLAN Traffic)                │   │
     │   └──────────────────────────────────────────┬────────────────────────────────────────────┘   │
     └──────────────────────────────────────────────┼────────────────────────────────────────────────┘
                                                    │ (802.1Q Trunk / 10G MikroTik CRS310 Spine)
                                                    ▼
     ┌───────────────────────────────── COMPUTATION & CLUSTER FABRIC ────────────────────────────────┐
     │                                                                                               │
     │   [ NODE 1: Intel 8505 Core ]                                                                 │
     │   ┌─────────────────────────────── VMID 200: App Core Container (VLAN 10) ────────────────┐   │
     │   │                                                                                       │   │
     │   │  ┌────────────────────────┐                   ┌────────────────────────┐              │   │
     │   │  │ Authentik Core & LDAP  │                   │ Twingate Connector     │              │   │
     │   │  │ • 10.10.10.10          │                   │ • 10.10.10.20          │              │   │
     │   │  │ • WebAuthn / OIDC/LDAP │                   │ • Scoped Resource Broker              │   │
     │   │  └───────────┬────────────┘                   └────────────────────────┘              │   │
     │   └──────────────│────────────────────────────────────────────────────────────────────────┘   │
     │                  │                                                                            │
     │                  │ (Internal LDAP Authentication Backend: port 389)                           │
     │                  ▼                                                                            │
     │   [ NODE 2: AMD Ryzen 5 Core ]                                                                │
     │   ┌──────────────┴──────────────── LXC 310: DMZ Media Cluster (VLAN 30) ──────────────────┐   │
     │   │                                                                                       │   │
     │   │  ┌────────────────────────────────────────────────────────────────────────┐           │   │
     │   │  │ Jellyfin Media Server (10.10.30.40)                                    │           │   │
     │   │  │ • Hardware NVENC Transcoding                                           │           │   │
     │   │  │ • Authentik LDAP Auth Integration (Global Directory Sync)              │           │   │
     │   │  │ • Server-Side Parental Controls (G to TV-MA Profile Gating)            │           │   │
     │   │  └────────────────────────────────────────────────────────────────────────┘           │   │
     │   └───────────────────────────────────────────────────────────────────────────────────────┘   │
     └───────────────────────────────────────────────────────────────────────────────────────────────┘
```

---

## 3. Engineering Implementation Details

### Tier 1: Caddy Edge & Authentik Identity Federation

Consumer smart TVs, streaming media appliances, and mobile apps cannot process interactive HTTP 302 redirects triggered by edge authentication portals.

- Edge Routing (OVHcloud VPS): Caddy terminates public TLS, enforces CrowdSec IPS remediation and GeoIP filtering, and routes traffic across an outbound-initiated WireGuard /30 tunnel directly to OPNsense.

- Web Application Ingress (Immich, Dashboards): Forward-Auth / OIDC policies challenge browser sessions with WebAuthn/Passkey MFA prior to granting access to APP_CORE (VLAN 10).

- Media Ingress (Jellyfin): Client traffic (jellyfin.domain.com) routes cleanly to DMZ_MEDIA (VLAN 30) without external HTML login redirects. Client applications (Neptune, Swiftfin, Android TV) query Jellyfin APIs directly.

- Identity Backend: Jellyfin delegates credential verification to an Authentik LDAP Outpost (ldap://10.10.10.10:389). User access is managed via the jellyfin_users group in Authentik, while granular content ratings (G through TV-MA), user profiles, and PIN locks are enforced at the Jellyfin server layer..

### Tier 2: Twingate Least-Privilege Administrative Access

- The Twingate Connector is deployed as an unprivileged container in **APP_CORE (VLAN 10)**.
- Point-to-point software-defined perimeters replace traditional Layer 3 VPN tunnels. Administrators access designated host:port definitions exclusively (e.g., 10.10.10.2:8006 for Proxmox VE), eliminating lateral network discovery and preventing untrusted subnet traversal.

### Tier 3: Tailscale Mesh with GL.iNet Hardware Integration

- OPNsense on the Intel 8505 gateway operates Tailscale as a kernel-accelerated Subnet Router.
- A portable GL.iNet Slate 7 Pro travel router acts as a mobile satellite node on 10.150.1.0/24. Client devices connected to the mobile travel Wi-Fi automatically resolve split-horizon DNS records (\*.internal.domain) through the secure mesh backhaul without requiring client-side VPN software on individual mobile endpoints.

### Tier 4: Internal Metasearch Microsegmentation (SearXNG)

Publicly exposing self-hosted metasearch engines invites automated scrapers and botnets that rapidly exhaust upstream quotas and trigger persistent IP blocks from providers like Google, Bing, and Brave.

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

## 4. Production Configuration Artifacts

### Caddy Edge Ingress Block (/etc/caddy/Caddyfile)

```text
# Public Media Streaming (Direct API Passthrough)
jellyfin.domain.com {
    request_body {
        max_size 20MB
    }

    # Proxy directly over WireGuard to OPNsense VIP
    reverse_proxy 10.10.30.40:8096 {
        header_up X-Real-IP {remote_host}
        header_up X-Forwarded-For {remote_host}
        header_up X-Forwarded-Proto {scheme}
    }
}

# Protected Internal Web Applications (Authentik Forward-Auth)
photos.domain.com {
    # Forward-Auth challenge against Authentik core
    forward_auth 10.10.10.10:9000 {
        uri /outpost.goauthentik.io/auth/caddy
        copy_headers X-Authentik-Username X-Authentik-Groups X-Authentik-Email
    }

    reverse_proxy 10.10.10.50:2283
}
```

# Authentik LDAP Outpost Provider Configuration (docker-compose.ldap.yml)

```text
services:
  authentik-ldap:
    image: ghcr.io/goauthentik/ldap:2024.8.3
    container_name: authentik_ldap_outpost
    restart: unless-stopped
    environment:
      AUTHENTIK_HOST: "[http://10.10.10.10:9000](http://10.10.10.10:9000)"
      AUTHENTIK_INSECURE: "false"
      AUTHENTIK_TOKEN: "${AUTHENTIK_OUTPOST_TOKEN}"
    ports:
      - "10.10.10.10:389:3389"
    networks:
      - internal_mgmt

networks:
  internal_mgmt:
    external: true
```

---
title: "NOC/SOC Defense: Packet Analysis, Threat Bouncing, and SIEM Telemetry"
date: 2026-09-30
draft: false
tags: ["CrowdSec", "Wireshark", "TCPDump", "Splunk", "DPI", "SIEM"]
summary: "Implementing a hybrid CrowdSec IPS and log analysis pipeline across edge proxies and hypervisors to detect, analyze, and drop malicious traffic flows."
weight: 5
---

## Overview

A comprehensive detection and incident response pipeline designed to capture suspicious network traffic, correlate telemetry across distributed log collectors, and automate perimeter remediation.

---

## 1. Real-Time Distributed Intrusion Prevention (CrowdSec)

Traditional Fail2ban deployments suffer from isolated local visibility. This architecture leverages a **Hybrid CrowdSec IPS Engine**:

- **Edge Log Acquisition:** CrowdSec parsers read real-time access logs from the OVHcloud edge proxy and home reverse proxy containers.
- **Central LAPI Coordination:** Container parsers push alerts to a centralized CrowdSec Local API (LAPI) engine.
- **Firewall Remediation Bouncers:** When brute-force attacks, port scans, or web exploit attempts are detected, the firewall bouncer injects temporary packet drop tables into OPNsense WAN rulesets—dropping malicious source IPs before packets traverse deeper into the network.

---

## 2. Packet Analysis & Anomaly Detection Workflows

### Protocol & Deep Packet Inspection

- **Zenarmor Layer 7 DPI:** Classifies evasive TLS flows and analyzes payload metadata across internal subnets to detect lateral traversal attempts between VLAN 30 (IoT) and secure zones.
- **Forensic Capture (TCPDump & Wireshark):** Automated packet capture scripts dump anomalous ingress flows matching specific TCP flags (SYN-flood patterns, non-standard TLS handshakes) into PCAP files for deep inspection in Wireshark.

### DNS Privacy & Tunnel Evasion

- Standard plaintext DNS (port 53) is intercepted and redirected via OPNsense NAT port forwards.
- Upstream resolution uses **DNS-over-QUIC (DoQ)** to Control D, backed by **Oblivious DoH (ODoH)** failover relays, stripping metadata from upstream ISP logging.

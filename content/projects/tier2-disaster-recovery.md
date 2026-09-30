---
title: "Two-Tier Disaster Recovery & Hardened Container Orchestration"
date: 2026-09-30
draft: false
tags: ["Kopia", "Backblaze-B2", "Disaster-Recovery", "Docker", "DevOps"]
summary: "Zero-plaintext container deployments paired with a two-tier backup pipeline utilizing local block snapshots and client-side encrypted Backblaze B2 offsite syncs."
weight: 3
---

## Engineering Overview

High-availability homelab operations require reproducible container deployments and an automated disaster recovery pipeline adhering strictly to the 3-2-1 backup standard.

---

## 1. Hardened Container Perimeter Standards

All container deployments adhere to strict runtime isolation boundaries deployed through GitOps orchestration:

- **Docker Named Secrets:** Plaintext passwords inside environment variables (`.env`) are prohibited. Secrets are mounted as secure file objects under Unix `chmod 600` permissions and injected into memory at `/run/secrets/`.
- **Runtime Sandboxing:**

  ```yaml
  read_only: true
  tmpfs:
    - /tmp:size=50M
  security_opt:
    - no-new-privileges:true
  cap_drop:
    - ALL

        ┌────────────────────────┐
        │ Production Hypervisors │
        └───────────┬────────────┘
                    │
          ┌─────────┴─────────┐
          │ Daily Sync Pipeline
          ▼                   ▼
  ┌──────────────────┐ ┌───────────────────────────┐
  │ Tier 1: Fast PBS │ │ Tier 2: Encrypted Offsite │
  │ (Local ZFS SSD)  │ │ (Kopia -> Backblaze B2)   │
  └──────────────────┘ └───────────────────────────┘
  ```

  Tier 1 (Rapid Local Recovery): Incremental, deduplicated Proxmox VE block snapshots streamed across 10G interconnects to an isolated Proxmox Backup Server (PBS) datastore residing on local enterprise ZFS SSD mirrors.

Tier 2 (Offsite Catastrophic Recovery): Daily automated snapshots of mission-critical datasets (/storage/photos, databases, password vault ledgers, Git configs) processed via Kopia.

    Client-Side Deduplication & Encryption: End-to-end AES-256-GCM encryption before data leaves local memory.

    Target: Scoped Backblaze B2 Cloud Object Storage (extra-infra-backup-kopia-daily-us-east).

    Bandwidth Optimization: Bulk, replaceable media streams are programmatically excluded to prevent cloud egress and storage overhead.

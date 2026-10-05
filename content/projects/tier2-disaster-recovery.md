---
title: "Two-Tier Disaster Recovery & Hardened Container Orchestration"
date: 2026-09-30
draft: false
tags: ["Kopia", "Backblaze-B2", "Disaster-Recovery", "Docker", "DevOps"]
summary: "Zero-plaintext container deployments paired with a two-tier backup pipeline utilizing local block snapshots and client-side encrypted Backblaze B2 offsite syncs."
weight: 6
---

## Engineering Overview

High-availability homelab operations require reproducible container deployments and an automated disaster recovery pipeline adhering strictly to the 3-2-1 backup standard.

---

## 1. Hardened Container Perimeter Standards

All container deployments adhere to strict runtime isolation boundaries deployed through GitOps orchestration:

- **Docker Named Secrets:** Plaintext passwords inside environment variables (`.env`) are prohibited. Secrets are mounted as secure file objects under Unix `chmod 600` permissions and injected into memory at `/run/secrets/`.
- **Runtime Sandboxing:**

```yaml
services:
  app:
    image: app:latest
    user: "1000:1000"
    read_only: true
    tmpfs:
      - /tmp:size=50M
    security_opt:
      - no-new-privileges:true
    cap_drop:
      - ALL
    secrets:
      - db_password

secrets:
  db_password:
    file: ./secrets/db_password.txt
```

```yaml
              ┌────────────────────────┐
              │ Production Hypervisors │
              └───────────┬────────────┘
                          │
                ┌─────────┴─────────┐
                │Daily Sync Pipeline│
                ▼                   ▼
      ┌──────────────────┐ ┌───────────────────────────┐
      │ Tier 1: Fast PBS │ │ Tier 2: Encrypted Offsite │
      │ (Local ZFS SSD)  │ │ (Kopia -> Backblaze B2)   │
      └──────────────────┘ └───────────────────────────┘
```

### Tier 1: Rapid Local Recovery

- **Mechanism:** Incremental, deduplicated Proxmox VE block snapshots streamed across 10G interconnects to an isolated Proxmox Backup Server (PBS) datastore.
- **Storage Plane:** Local enterprise ZFS SSD mirrors providing near-instantaneous recovery times for hypervisor VMs and LXCs.

### Tier 2: Offsite Catastrophic Recovery

- **Mechanism:** Daily automated snapshots of mission-critical datasets (`/storage/photos`, cryptographic state files, application databases, and Git configs) processed via the Kopia Engine.
- **Database Integrity:** Stateful transactional systems execute automated database dumps to a staging directory prior to Kopia snapshot execution to prevent state corruption.
- **Client-Side Encryption:** End-to-end AES-256-GCM encryption is enforced locally before data leaves host memory.
- **Target Bucket:** Scoped Backblaze B2 Cloud Object Storage (`extra-infra-backup-kopia-daily-us-east`).
- **Bandwidth Optimization:** Bulk, replaceable media streams are programmatically excluded to mitigate cloud storage overhead and unexpected data egress costs.

---

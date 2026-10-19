---
layout: '../../layouts/Project.astro'
title: NAS Homelab
description: "Containerized media stack, automated backup pipelines, and secure reverse-proxy routing."
badge: "Self-Hosted"
tags: ["Docker", "Tailscale", "Caddy"]
---

## The Core Philosophy

At some point, paying recurring subscription fees to let third-party tech conglomerates compress family vacation photos and train models on text documents becomes philosophically irritating. 

The rational human response is to pay an order of magnitude more upfront for dedicated hardware, configure network routing tables until 3:00 AM, and assume full operational liability for personal uptime. 

This project chronicles the migration from "the cloud is just someone else's computer" to "someone else's computer is now drawing 45 watts next to my desk."

---

## The Hardware Rig

The foundation is built on a **UGREEN NAS enclosure** (*DXP4800 Pro*) running **UGOS Pro**. It does not look like an enterprise rack unit ready to cool down a server room; it looks like a polite aluminum toaster that quietly handles several terabytes of mirrored solid-state and mechanical storage.



```
              ┌─────────────────────────────────────────┐
              │             UGREEN NAS Unit             │
              │             (OS: UGOS Pro)              │
              └────────────────────┬────────────────────┘
                                   │
            ┌──────────────────────┴──────────────────────┐
            ▼                                             ▼
 [ Tailscale WireGuard ]                       [ Caddy Reverse Proxy ]
            │                                             │
┌───────────┴────────────┐                  ┌────────────┴─────────────┐
▼                         ▼                  ▼                           ▼
Immich (Photos)      Nextcloud (Files)   Directus (Databases)       Beszel / Dozzle

```

---

## The Software Stack & Services

UGOS Pro handles the underlying storage pools, but the actual work runs inside containerized environments managed via Docker Compose. 

### 1. Photo Preservation: Immich
Google Photos is convenient until storage thresholds trigger passive-aggressive storage upgrade banners. 

* **The Replacement:** A self-hosted [Immich](https://immich.app/) instance.
* **The Reality:** Handles machine-learning facial recognition, geospatial mapping, and automated background sync from mobile devices without sending raw telemetry to Mountain View. 
* **The User Base:** Successfully onboarded immediate family members. If the photos stop syncing during Thanksgiving, I am the tier-1 support engineer.

### 2. File Systems & Data Governance: Nextcloud
* Handles cross-device file synchronization, calendar sync, and general cloud utility replacement.
* Paired with offsite snapshotting because RAID is a high-availability measure, not a backup strategy. If you do not test your restores, you do not have backups; you have hope.

### 3. Structured Data: Directus
* Serves as a headless CMS and structured database wrapper.
* Currently stores cataloged schemas, asset metadata, and custom inventories (including personal collections and physical item tracking) via an intuitive REST/GraphQL layer.

### 4. Telemetry & Observability
Because running headless software without metrics is just waiting for a disk to silently fill up until a process crashes:
* **Beszel:** Lightweight, no-nonsense server telemetry tracking CPU spikes, RAM allocation, and ambient temperature.
* **Dozzle:** Real-time container log viewer. Far superior to running `docker logs -f --tail 100 <container_id>` inside an SSH session on a phone.

---

## Networking: How to Not Expose Everything to the Internet

Opening ports on a residential ISP router is a time-tested strategy for learning how fast botnets scan IPv4 subnets. We do not do that here.


```
Client Device (Phone / Laptop)
│
▼
[ Tailscale Mesh VPN ]
│  (Encrypted WireGuard Tunnel)
▼
[ Caddy Reverse Proxy ] ───► SSL Termination (Internal certs)
│
├──► immich.internal:2283
├──► cloud.internal:8080
└──► dozzle.internal:8888
```

1. **Tailscale:** Every device (laptops, phones, remote clients) communicates over an encrypted WireGuard mesh network. To the outside world, zero ports are open to the open internet.
2. **Caddy:** Operates as the internal reverse proxy. Handles internal routing, SSL termination, and readable local domain aliases (`*.internal`) so nobody has to remember random 5-digit port numbers.
3. **Cloudflare Tunnels:** Reserved strictly for external endpoints that need authenticated zero-trust handshakes without poking holes in the firewall.

---

## Continuous Deployment via GitHub Actions

Updating the static frontend portfolio (built on Astro) does not involve dragging files over SFTP like it is 2004.

```yaml
name: Deploy to Self-Hosted NAS
on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout Source
        uses: actions/checkout@v4

      - name: Connect to Mesh
        uses: tailscale/github-action@v2
        with:
          authkey: ${{ secrets.TAILSCALE_AUTHKEY }}

      - name: Build & Rsync Static Payload
        run: |
          npm ci
          npm run build
          rsync -avz --delete ./dist/ deploy@nas.internal:/var/www/osquijrdev/
```

Every `git push` to `main` spins up an ephemeral runner, authenticates onto the private Tailscale tailnet using an ephemeral auth key, compiles the Astro production bundle, and rsyncs the build artifact directly into the web root on the NAS.

If a build fails, it fails in GitHub's container, not in production.

---

## Retrospective & Current State

| Metric | Status |
| --- | --- |
| **Monthly Subscription Fees Eliminated** | ~$25/month |
| **Hardware Payback Period** | Somewhere around late 2029 |
| **Silent Fault Tolerance** | High (Mirrored arrays + automated snapshots) |
| **Peace of Mind** | 100% data ownership, 0% dependency on third-party cloud policies |

The infrastructure now quietly hums along, backing up files, tracking metrics, and hosting this website. It turns out that when you build things correctly the first time, you don't actually have to spend every weekend fixing them.


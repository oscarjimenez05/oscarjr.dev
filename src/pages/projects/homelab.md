---
layout: '../../layouts/Project.astro'
title: NAS Homelab
description: "Containerized media stack, automated backup pipelines, and secure reverse-proxy routing."
badge: "Self-Hosted"
tags: ["Docker", "Tailscale", "Cloudflare"]
---

## The Core Philosophy
I was running out of storage, so I bought an 8tb Hard Drive that I told myself I would back up all of my files to on a weekly basis. That exercise lasted less than two weeks, so I needed a more long-term solution.

Since I don't like recurring fees the next best alternative was to pay an order of magnitude more upfront for dedicated hardware, work around campus network restrictions, and assume full operational liability for personal and family uptime. 

This is how this all worked out, and what important skills I learned from this project. I will try my best to be descriptive without revealing details that will get me hacked.

---

## The Hardware Rig
The foundation is built on a **UGREEN NAS enclosure** (*DXP4800 Pro*) running **UGOS Pro** (Debian-based OS). This little toaster is running two separate RAID 1 HDD volumes, plus two NVMes for the operating system and container data. 

Normally, one would be able to connect to this through the NAS's static IP address if on the same Local Area Network. It was devastating to find out that for security purposes, the Residential Network of Ithaca College does not allow this direct communication. So there's a stack of solutions in place:

### UGREENLink
A relay connection through UGREEN's servers, so really slow and not useful for any bandwidth-heavy or latency-sensitive task. Additionally, can only practically access the raw **UGOS Pro** through it, services on specific ports become unaccessible. Also exposed to public internet, so required strong rotating passwords and 2FA authentication. This is still up but only for when the methods below fail (most commonly when a credential expires).

### Tailscale
Mesh VPN connection that allows for P2P communication between end hosts and NAS, even through restrictive college network. When that direct connection fails still resorts to Tailscale's relay servers, but they are still faster than UGREEN's. Plus I have managed to get direct connections even across international borders. Truly life-saving technology.

### Cloudflare Tunnel
*Most* public services pass through this tunnel network.

---

## Services
UGOS Pro handles the underlying storage pools, but the actual work runs inside containerized environments managed via Docker Compose.

### Public Services
- Small photo sharing container using FileBrowser Quantum.
- Song review system using the freeframe.io software.
- Self-hosted Overleaf instance for collaboration after running out of free plan compile limits for [this research project](/projects/lehmerprng) on the cloud version.

#### Media Library

One additional public service is my personal Jellyfin media collection. Because Cloudflare Tunnel's terms of service explicitly prohibit heavy media streaming, I had to look for an alternative solution. So I booted up a disposable Oracle Cloud virtual machine that is connected to the same *tailscale* network as the NAS. So clients talk to this public machine outside Ithaca College restrictions which then relays to my NAS through *tailscale*.

However, because this VM has to be set to DNS only in the Cloudflare dashboard, I do not have built-in DDoS protection, so I had to install fail2ban in it. Additionally, had to set permissions so it can only talk to the NAS in the *tailscale* mesh, and only on the specific port for Jellyfin. This is to prevent some sort of mesh-wide attack in case there was a vulnerability in Caddy that allowed to make arbitrary requests to anything else.

### Private Services
#### Photo Library
Google Photos is convenient until storage thresholds trigger passive-aggressive storage upgrade banners. Or when they increase their prices, or shut down features without warning, or when they add gen-AI bloat where it doesn't make sense, or when they use your sensitive photos to train said gen-AI bloat.

- **The Replacement:** A self-hosted [Immich](https://immich.app/) instance.
- **The Reality:** Handles machine-learning facial recognition, geospatial mapping, and automated background sync from mobile devices without sending raw telemetry to Mountain View. 
- **The User Base:** Successfully onboarded immediate family members. If the photos stop syncing during Thanksgiving, I am the tier-1 support engineer.

> Currently working on a database project that pulls instance data and augments it to get deeper photo insights. Coming soon on the [projects](/projects) page.

#### File System
- Nextcloud cross-device file synchronization, calendar sync, and general cloud utility replacement.
- Paired with offsite snapshotting because RAID is a high-availability measure, not a backup strategy. If you do not test your restores, you do not have backups; you have hope.

#### Database
- Using Directus as a headless CMS and structured database wrapper.
- Currently stores cataloged schemas, asset metadata, and custom inventories (including personal mineral collections and physical item tracking) via an intuitive REST/GraphQL layer. Will integrate to this website soon!

#### Monitoring
Running a couple of useful monitoring tools as well.
- **[Beszel](https://beszel.dev/):** Lightweight, no-nonsense server telemetry tracking CPU spikes, RAM allocation, and ambient temperature, and ICMP network monitors. Webhook connected to alerts on my phone.
- **[Dozzle](https://dozzle.dev):** Real-time container log viewer. Far superior to running `docker logs -f --tail 100 <container_id>` inside an SSH session on a phone everytime something goes wrong.

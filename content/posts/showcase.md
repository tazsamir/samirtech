---
title: "Showcase"
date: 2026-09-10T17:00:00+01:00
draft: false
url: "/showcase/"
description: "A practical homelab built from quiet, second-hand and low-power hardware."
summary: "The hardware, services and network design behind my homelab."
---

# My homelab

This is my personal lab: a small collection of repurposed hardware, storage and networking equipment used to learn, host services and keep important data under my control. It is intentionally practical rather than flashy, with low power use, recoverability and simple maintenance as the priorities.

## The lab at a glance

A logical map of the lab, not a rack photograph or a physical wiring plan. The everyday services share the home network; backup destinations are shown separately so they are not mistaken for machines that must all stay on.

### Everyday use

```text
      PC / phone / tablet / TV
                  |
          HOME NETWORK
                  |
   +--------------+--------------+
   |                             |
GEEKOM N100                 MAIN TRUENAS
Always-on Docker           HP desktop
   |                             |
   +-- Caddy: local HTTPS        +-- Files
   +-- CoreDNS: local names      +-- Photos
   +-- Hosted services          +-- Live storage
   +-- Monitoring
```

The N100 runs services; the main NAS holds the primary storage. They are peers on the network, not a chain where all NAS traffic passes through the N100.

### Recovery paths

```text
MAIN TRUENAS
   |
   +-- Local replication
   |       |
   |       v
   |   SECOND TRUENAS
   |   Fractal Node 804
   |   Usually off / suspended
   |
   +-- Encrypted cloud copy
           |
           v
       OFF-SITE STORAGE

SELECTED FILES + CONFIG COPIES
   |
   +-- Encrypted Restic backup
           |
           v
       RASPBERRY PI 4
       Separate backup destination
```

The second NAS is a local recovery copy, not an off-site backup. The Pi has passed sample file restores locally; placing it at a separate location and verifying remote connectivity remain separate steps. The Restic selection is not a full copy of every service, database or media library.

### On-demand experiments

```text
DELL MICRO PC
   +-- Proxmox / virtual machines
   +-- Powered off until needed

GL.iNet ROUTER
   +-- Isolated / mobile lab work
   +-- Separate from home-network edge
```

**Design goal:** keep daily services quiet and low-power, wake the experiment hardware only when needed, and keep recovery copies separate from the systems they protect.

## Hardware

### Fractal Node 804

The Node 804 is my TrueNAS Voyager. It is the second NAS and normally stays powered off or suspended, receiving local replication from Normandy for recovery and protected copies.

### GEEKOM N100

The N100 is the quiet, efficient always-on Docker host. It runs the lightweight services that do not need a full server, including the home proxy, DNS and monitoring tools.

### HP Enterprise desktop — TrueNAS Normandy

The HP Enterprise desktop is my main TrueNAS server, Normandy. It is housed in a two-bay enclosure and runs the live storage, photo library and primary file workloads.

### Dell Micro PC — Proxmox

The Dell Micro PC runs Proxmox for virtual machines and experiments. It is currently powered off, so it is available when needed without adding to the always-on power draw.

### GL.iNet router

The GL.iNet router is the network edge for isolated or mobile lab work. It lets me test network changes without exposing the rest of the home network.

### Raspberry Pi 4 — additional backup destination

The Pi holds an encrypted Restic repository for selected important files and copied configuration snapshots. A picture and Docker-project files have been restored and verified from it. It is intended for a separate location; remote-location connectivity is not established by those local tests.

[Read the implementation and restore evidence](/posts/using-a-second-nas-and-an-off-site-raspberry-pi/).

## How it fits together

The main PC and lab machines connect through the home network. The N100 publishes friendly local HTTPS names through Caddy and provides DNS for the services. The NAS stores the important data, while the second NAS, removable storage and encrypted off-site copies provide recovery paths.

This is a living setup. Hardware and services change as I learn, but the design goal stays the same: useful services, clear failure boundaries and backups that can actually be restored.

## Coming next

I will add photographs of each machine, power measurements and a service-by-service inventory as the lab documentation grows.

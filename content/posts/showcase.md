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

![Homelab topology diagram](/homelab-topology.svg)

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

## How it fits together

The main PC and lab machines connect through the home network. The N100 publishes friendly local HTTPS names through Caddy and provides DNS for the services. The NAS stores the important data, while the second NAS, removable storage and encrypted off-site copies provide recovery paths.

This is a living setup. Hardware and services change as I learn, but the design goal stays the same: useful services, clear failure boundaries and backups that can actually be restored.

## Coming next

I will add photographs of each machine, power measurements and a service-by-service inventory as the lab documentation grows.

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

The Node 804 is the main storage chassis. Its compact cube layout gives me room for multiple drives while keeping the system suitable for a home office. It is used for TrueNAS storage and the primary photo and file workloads.

### GEEKOM N100

The N100 is the quiet, efficient always-on Docker host. It runs the lightweight services that do not need a full server, including the home proxy, DNS and monitoring tools.

### HP Enterprise PC

The HP Enterprise PC is a repurposed workstation for heavier or experimental workloads. It gives the lab a place for virtual machines and tests without putting the always-on services at risk.

### Two-bay drive enclosure

The two-bay enclosure provides removable or secondary storage for protected copies, migrations and recovery work. It is useful precisely because it is separate from the main NAS.

### Dell Micro PC

The Dell Micro PC is another small, low-power machine for experiments and supporting services. Small business hardware like this is inexpensive to run and easy to replace.

### GL.iNet router

The GL.iNet router is the network edge for isolated or mobile lab work. It lets me test network changes without exposing the rest of the home network.

## How it fits together

The main PC and lab machines connect through the home network. The N100 publishes friendly local HTTPS names through Caddy and provides DNS for the services. The NAS stores the important data, while the second NAS, removable storage and encrypted off-site copies provide recovery paths.

This is a living setup. Hardware and services change as I learn, but the design goal stays the same: useful services, clear failure boundaries and backups that can actually be restored.

## Coming next

I will add photographs of each machine, power measurements and a service-by-service inventory as the lab documentation grows.

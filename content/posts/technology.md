---
title: "Technology"
description: "The tools and technologies behind my workstation, homelab and website."
layout: "single"
slug: "technology"
aliases: ["/technology/"]
draft: false
---

This is a high-level overview of the technology I use. It intentionally leaves out private addresses, credentials and details that would make the infrastructure less safe.

## Core systems

- **Linux and KDE** for the main workstation and daily technical work.
- **TrueNAS** for storage, datasets, replication and recovery testing.
- **Docker** for services that need repeatable deployment and bounded maintenance.
- **Low-power hardware** where quiet operation and efficient running matter.

## Services and networking

- **Caddy** for local HTTPS and reverse proxying.
- **CoreDNS** for friendly internal names.
- **Restic and encrypted archives** for configuration and recovery data.
- **Monitoring and health checks** for service state, backups and container changes.

## Publishing

- **Hugo** generates this site.
- **GitHub** stores the source and validates each build.
- **Cloudflare Pages** publishes the production site.

The tools matter less than the habits around them: least privilege, backups, updates, logs, documentation and restore tests.

---
title: "How My Restic Backup Protects the Homelab"
date: 2026-09-14T08:30:00+01:00
draft: false
description: "What my Restic backup covers, what it does not cover, and why configuration backups are still valuable during a recovery."
tags: [backups, restic, homelab, recovery]
series: ["Learning Backups the Hard Way"]
series_order: 8
---

Restic is one layer of my backup plan. It is not the backup for every file on every server, and understanding that boundary matters.

## What this backup is for

My Restic job protects the configuration needed to rebuild the surrounding homelab. That includes container definitions, stack files, service configuration and selected system settings such as scheduled jobs, mounts, remote-access configuration and security-related configuration.

The purpose is simple: if the host has to be rebuilt, I should not have to remember how every service was assembled. I can recover the documented configuration, reinstall the software and bring the services back in a controlled order.

## What it does not replace

This Restic backup is not a substitute for the Immich backup. Immich photos and videos, its PostgreSQL data and its application recovery material need to be protected as a related recovery set through their own backup layers.

That distinction prevents a common mistake: seeing a successful configuration backup and assuming the important application data is included. A backup only protects the sources it actually reads.

## Why Restic is useful

Restic provides encrypted, deduplicated, incremental backups. After the first run, later runs can reuse data already stored and upload only changed content. Encryption means the repository can be held on storage that is not itself trusted with readable copies of the configuration.

The repository password is therefore part of the backup. Without it, the encrypted data is not a recovery plan. I keep the password and the recovery notes separately from the machine being backed up, with more than one protected copy.

## Recovery is the test

A completed job is only evidence that a backup process ran. It does not prove that the repository can be opened or that the files needed for a rebuild are present.

A useful test is to restore selected configuration into a temporary area, inspect the files, and confirm that the deployment notes are sufficient to identify the services, storage relationships and required secrets. I must not restore over live configuration while testing.

For a real rebuild, I would restore the configuration first, reinstall the required applications, recover secrets through the protected process, and then restore each service's data from the backup designed for that service.

## The lesson

Restic is valuable because it preserves the instructions and configuration around the data. It reduces rebuild time and removes guesswork, but it works best as one clearly labelled layer beside the Immich media backup, database backup, TrueNAS snapshots and encrypted off-site copy.

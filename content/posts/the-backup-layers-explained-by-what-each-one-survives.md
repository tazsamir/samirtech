---
title: "The Backup Layers, Explained by What Each One Survives"
date: 2026-09-22T00:00:00+01:00
description: "A reference of every backup layer in my homelab, what data it protects, how fresh it is, and which failure each one actually recovers from."
tags: [backups, homelab, restic, truenas, 321-backup]
---

A backup design fails when you cannot say, for any given piece of data, *which copy protects it, how old that copy is, what unlocks it, and what has been recovered from it*. This is my attempt to write all four answers down in one place.

The point of listing layers is not to collect destinations. It is to stop pretending that several vague "backups" add up to safety when they might all protect different things, or the same thing at different ages, or nothing that has ever been tested.

## The layers

| Layer | What it protects | Freshness | Survives |
|---|---|---|---|
| Immich live data | Original media + PostgreSQL database | Real time | Nothing — it is the thing being protected |
| TrueNAS snapshots | Short-term rollback of the photo library | Hourly, retained two weeks | Accidental deletion or bad edit on the same storage |
| TrueNAS cloud-sync | Encrypted copy of Immich data to B2 | Daily, with a source snapshot | Loss of the whole NAS |
| Server configuration backup | Docker host config and selected app data to a Restic repository | Weekly (Sunday 04:15) | Broken host or container config, not the media itself |
| Off-site Pi backup | Selected irreplaceable files to an encrypted Restic repo | Daily (06:15); a successful scheduled post-relocation run was documented on 12 October 2026 | Loss of the house; the Pi is now physically at a separate location |
| Second NAS (Borgcube) | Local recovery destination | Powered off when idle | Fast local recovery, not a scheduled guarantee |

Two caveats keep this list honest:

- **The second NAS** is a recovery destination I have used during restores, but it is normally powered off or suspended. Local replication *should* be part of the design, but I do not describe it as a current scheduled guarantee without checking its enabled tasks, scope and latest successful run.
- **The Pi is physically off-site and remotely reachable.** Scheduled backups have succeeded after relocation; the latest documented run on 12 October 2026 reported 46 snapshots, and a repository integrity check completed without errors. A picture and Docker-project files had previously been restored and verified locally from the Pi. Those are distinct checks: location, connectivity, a scheduled backup, repository integrity and a sample file restore do not amount to a full replacement-host or application-and-database recovery.

## What each layer is *not*

Freshness is not the same as independence. Copying a weekly snapshot daily does not create daily recovery points — it moves an already-old snapshot more often. Each Pi snapshot group has its own schedule and source freshness, so a fresh run for one group says nothing about another.

Encryption is not immutability. An encrypted repository still allows an authorized client to delete or damage it. Encryption protects a copy from being read, not from being destroyed.

A passing repository integrity check is useful evidence about the repository structure, but it is not a full restore test. A successful scheduled run proves the job completed; a sample file restore proves only the selected files could be recovered; neither proves that database dumps are consistent or that a replacement host can run every application.

## How to read this correctly

The one sentence that matters: **which copy protects which data, how old it is, what key opens it, and whether a restore from it has actually succeeded.** Any backup story that cannot answer all four is a hope, not a plan.

The layers above are my current honest answer, with their limits kept explicit rather than smoothed over. That is the whole discipline: keep the gaps visible, so the next recovery test targets them instead of assuming they are already handled.

For the practical checks and recovery sequence, see [My Post-Backup Verification Checklist](/posts/post-backup-verification-checklist/), [Restoring an Application from Backup](/posts/restoring-an-application-from-backup/) and [Using a Second NAS and an Off-Site Raspberry Pi](/posts/using-a-second-nas-and-an-off-site-raspberry-pi/).
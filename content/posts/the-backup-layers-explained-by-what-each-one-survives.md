---
title: "The Backup Layers, Explained by What Each One Survives"
date: 2026-09-22T00:00:00+01:00
draft: true
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
| Off-site Pi backup | Selected irreplaceable files to an encrypted Restic repo | Daily (06:15) | Loss of the house, assuming the Pi is elsewhere |
| Second NAS (Voyager) | Local recovery destination | Powered off when idle | Fast local recovery, not a scheduled guarantee |

Two layers do **not** belong on this list yet:

- **The second NAS** is a recovery destination I have used during restores, but it is normally powered off or suspended. Local replication *should* be part of the design, but I do not describe it as a current scheduled guarantee without checking its enabled tasks, scope and latest successful run.
- **The Pi's remote-location claim** is untested. The selected-data backup has completed and a picture and Docker files were restored from it, but the Pi has not been relocated, so "off-site" describes the intention, not a verified state.

## What each layer is *not*

Freshness is not the same as independence. Copying a weekly snapshot daily does not create daily recovery points — it moves an already-old snapshot more often. The Pi `parents-irreplaceable` snapshot succeeding on 22 September does not make the `server-config` snapshot newer than its 20 September run; they are separate tags, separate schedules, separate data.

Encryption is not immutability. An encrypted repository still allows an authorized client to delete or damage it. Encryption protects a copy from being read, not from being destroyed.

A passing repository check is not a full restore test. Getting the snapshot list, or the `state: success` line in a status file, proves the job ran — not that the data is recoverable, not that the database dumps are consistent, not that the passphrase can actually unlock it.

## How to read this correctly

The one sentence that matters: **which copy protects which data, how old it is, what key opens it, and whether a restore from it has actually succeeded.** Any backup story that cannot answer all four is a hope, not a plan.

The layers above are my current honest answer, with the two unverified labels kept explicit rather than smoothed over. That is the whole discipline: keep the gaps visible, so the next recovery test targets them instead of assuming they are already handled.
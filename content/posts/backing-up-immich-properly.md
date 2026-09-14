---
title: "Backing Up Immich Properly: Photos, Database and Encryption Keys"
date: 2026-09-14T08:00:00+01:00
draft: false
description: "What I learned about backing up an Immich installation after rebuilding one from recovered media and database data."
tags: [immich, backups, truenas, databases]
series: ["Learning Backups the Hard Way"]
series_order: 4
---

Immich makes a self-hosted photo library feel simple, but the backup is more than the visible photo folders.

My recovery showed that I needed to protect more than the visible photo library. I need to protect the original media, the PostgreSQL database, the Immich application configuration and the credentials or keys required to access the backup.

## Original media

The originals are the irreplaceable part. They include photographs and videos uploaded over many years, including files that are no longer on current phones or computers.

I now treat the Immich library as ordinary important data as well as application data. It needs copies that can be opened outside Immich, not only thumbnails that appear in the web interface.

## The database

The database contains users, albums, metadata and the relationships between assets. Restoring only the media can recover the files, but it does not recreate the library experience.

During recovery I created a PostgreSQL dump and checked the restored database. I also kept the restored Immich installation available as a source while migrating into the active server.

The database must be backed up separately from the media. A dump that has never been restored is an assumption, not proof.

## How the backup layers fit together

The live Immich installation stores the original media and PostgreSQL data as separate parts of the system. They must be treated as one recovery set, even though they are backed up differently.

Local TrueNAS copies provide a nearby recovery route when the primary storage or an individual service fails. They are useful for fast recovery, but they are not protection from every failure: both systems may be affected by the same mistake, physical event or missing encryption key.

## Snapshots and cloud sync

The Immich storage has an enabled recursive TrueNAS snapshot task. It runs hourly and retains snapshots for two weeks. The snapshots use a dedicated naming pattern, making them easier to identify during recovery.

Snapshots are useful for accidental deletion, corruption or a recent change, but they are not a complete backup. They remain on the same storage system and can be lost with the pool, hardware or encryption key. They also do not replace a tested database restore.

There is also an enabled encrypted cloud-sync task for Immich. It pushes a copy to Backblaze B2 each day and creates a source snapshot as part of the transfer. This gives the photo library an off-site copy while keeping the transfer separate from the local snapshot schedule.

A separate cloud-sync restore test exists but is currently disabled. That is an important distinction: having a configured off-site backup is not the same as regularly proving that it can be restored. I need to enable or manually perform a controlled restore test, using a separate recovery location and without writing over the live library.

The off-site copy is encrypted and stored in Backblaze B2 through the TrueNAS cloud-sync task. It is not a normal folder that can simply be browsed; recovery requires the correct protected configuration and encryption credentials before it produces ordinary recovered files.

The wider server configuration is backed up separately with Restic. That backup helps rebuild the host, containers and supporting services, but it is not a substitute for backing up the Immich originals and PostgreSQL data.

## Encryption keys and passphrases

My recovery involved two different encryption issues. Some TrueNAS-replicated datasets could not be opened because the required TrueNAS key was missing. The off-site cloud copy was recoverable because I still had the required encryption credentials.

The lesson is straightforward: store dataset keys, backup passphrases, B2 credentials and administrator recovery details outside the NAS. Keep more than one protected copy and make sure the recovery instructions explain which key belongs to which system.

## A practical recovery set

For each Immich backup, I want the following available:

- the original media;
- a recent PostgreSQL dump;
- the Immich version and deployment notes;
- configuration and environment details without exposing secrets publicly;
- TrueNAS dataset keys;
- B2 or backup-provider details;
- the encryption passphrase;
- a written restore procedure.

## Verification

I check photos and videos from different years, open originals outside Immich and compare selected files with SHA-256 hashes. I also check the Immich timeline and retry missing periods rather than assuming that a successful migration summary means everything arrived.

The result is a backup that can be rebuilt as a service and also accessed as a collection of normal files if Immich itself is unavailable.


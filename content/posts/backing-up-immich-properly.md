---
title: "backing-up-immich-properly"
date: 2026-09-19T09:00:00+01:00
draft: false
series: ["Learning Backups the Hard Way"]
schema: 2
---

---
title: "Backing Up Immich Properly: Photos, Database and Encryption Keys"
date: 2026-09-13T09:00:00+01:00
draft: true
description: "What I learned about backing up an Immich installation after rebuilding one from recovered media and database data."
tags: [immich, backups, truenas, databases]
series: ["Learning Backups the Hard Way"]
series_order: 4
---

Immich makes a self-hosted photo library feel simple, but the backup is more than the visible photo folders.

My recovery showed that I needed to protect three related things: the original media, the PostgreSQL database and the credentials/keys required to access the backup.

## Original media

The originals are the irreplaceable part. They include photographs and videos uploaded over many years, including files that are no longer on current phones or computers.

I now treat the Immich library as ordinary important data as well as application data. It needs copies that can be opened outside Immich, not only thumbnails that appear in the web interface.

## The database

The database contains users, albums, metadata and the relationships between assets. Restoring only the media can recover the files, but it does not recreate the library experience.

During recovery I created a PostgreSQL dump and checked the restored database. I also kept the restored Immich installation available as a source while migrating into the active server.

The database backup must be scheduled, retained and tested. A dump that has never been restored is an assumption, not proof.

## Encryption keys and passphrases

My recovery involved two different encryption issues. Some TrueNAS-replicated datasets could not be opened because the required TrueNAS key was missing. The Backblaze copy was recoverable because I still had the Duplicati encryption passphrase.

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


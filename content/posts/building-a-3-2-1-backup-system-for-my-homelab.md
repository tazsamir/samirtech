---
title: "Building a 3-2-1 Backup System for My Homelab"
date: 2026-09-12T12:10:00+01:00
lastmod: 2026-09-16T03:00:00+01:00
draft: false
description: "How local snapshots, encrypted cloud backups and a Raspberry Pi fit together, and which recovery tests have actually passed."
tags: [backups, homelab, truenas, 321-backup]
series: ["Learning Backups the Hard Way"]
series_order: 3
---

After recovering my photos, I needed a backup design that did not depend on luck. The familiar 3-2-1 rule gives me a useful structure:

- **3 copies** of important data, including the working copy;
- **2 different media or storage types**, rather than every copy depending on the same storage system;
- **1 copy off-site**, outside the same physical failure zone.

The numbers are a starting point, not proof of recoverability. Another folder on the same NAS is not an independent backup, and another machine in the same house is not an off-site copy.

## What is working now

**Documentation updated: 16 September 2026.** This summary reflects the completed checks recorded in the linked articles and the 15 September configuration-backup check. It is not a new full recovery test.

- **Immich:** original media and PostgreSQL data are separate parts of the live service. Both matter to recovery.
- **Local snapshots:** the documented Immich snapshot task runs hourly and retains snapshots for two weeks. These provide rollback on the same storage system, not protection from losing that system.
- **Encrypted cloud backup:** the documented daily TrueNAS cloud-sync task pushes Immich data to Backblaze B2 and creates a source snapshot. An encrypted cloud recovery test completed successfully.
- **Host configuration:** a weekly Restic job backs up the Docker host's configuration and selected application data to TrueNAS. The destination mount, repository structure and retrieval of a backed-up Compose file were checked.
- **Raspberry Pi:** the initial selected-data backup completed, and a picture and Docker-project files were restored and verified from its encrypted Restic repository.

The Pi is intended for a separate location. The completed tests do **not** establish connectivity after relocation. Calling it an off-site design must not imply that the remote-location test has already happened.

## How the layers fit together

```text
Live photo service
  ├─ Original media + PostgreSQL data
  ├─ Local snapshots: short-term rollback on the same storage
  └─ Encrypted cloud copy: an independent off-site recovery route

Docker host
  └─ Weekly configuration/application-data snapshots → TrueNAS Restic repository

Selected NAS files + existing configuration snapshots
  └─ Daily Pi job → encrypted Restic repository
                     remote-location connectivity still to be tested
```

These branches do not all contain the same data. A successful host-configuration backup is not evidence that the entire photo library or a fresh, consistent database dump is included. The Pi copies existing configuration snapshots; it does not make those snapshots newer.

## The second NAS has a separate role

A second TrueNAS machine provides a local recovery destination and has been used during recovery. It normally stays powered off or suspended. Local replication belongs in the design, but I should not describe it as a current scheduled recovery guarantee without checking its enabled tasks, scope and latest successful runs.

Two NAS machines in one home still share risks: fire, theft, power problems and mistakes made with the same administrator access. Replication also needs an appropriate retention policy; propagating a change is not the same as preserving a recoverable older version.

## Keys are part of the backup

The recovery taught me that dataset keys and backup passphrases must be stored separately from the data they unlock. Protected copies belong in more than one secure place, including a recovery route that does not depend on the failed server.

Removable key storage is not another full data backup. It holds information required to unlock and rebuild the actual backups. Repository encryption also does not make a destination immutable: an authorized client may still be able to delete or damage it.

## What the tests prove—and what they do not

The completed cloud recovery and Pi sample restores are useful evidence. They show that those tested copies could be accessed, decrypted and recovered at the time of the tests.

They do not establish a complete replacement-server recovery, a fresh end-to-end Immich database restore from every backup destination, or reliable operation after moving the Pi. A structural repository check is also different from reading every stored data block.

My next confidence steps are an isolated application/database rehearsal, a remote-location connectivity test and deliberate review of Pi retention and capacity. The Pi setup currently has no automatic off-site pruning.

## Read the implementation details

- [Backing up Immich properly](/posts/backing-up-immich-properly/) explains the media, database and key dependencies.
- [How my Restic backup protects the homelab](/posts/how-my-restic-backup-protects-the-homelab/) explains scope and backup freshness.
- [Using a second NAS and an off-site Raspberry Pi](/posts/using-a-second-nas-and-an-off-site-raspberry-pi/) records the implementation and actual restore evidence.
- [Testing a restore before disaster strikes](/posts/testing-a-restore-before-disaster-strikes/) describes the checks that turn a backup into a usable recovery path.

The goal is not to collect more destinations. It is to know which copy protects which data, how old it is, what unlocks it and what has actually been recovered from it.

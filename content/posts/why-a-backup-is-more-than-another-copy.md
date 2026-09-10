---
title: "Why a Backup Is More Than Just Another Copy"
date: 2026-09-11T09:00:00+01:00
draft: false
description: "Recovering my family photos taught me that a backup is only useful when it can be unlocked, restored and verified."
tags: [backups, homelab, data-recovery]
series: ["Learning Backups the Hard Way"]
series_order: 2
---

Before I lost my main TrueNAS system, I thought I had backups. In a narrow sense, I did. There were replicas, cloud data and older copies scattered across different machines.

The problem was that I had not proved I could recover from them.

## A copy is not automatically a backup

A second copy can still fail as a backup if:

- it is encrypted and the key is missing;
- it depends on software or a database you no longer have;
- you do not know which copy is complete;
- it has never been restored and tested;
- it is stored beside the original and can be destroyed at the same time;
- the credentials or passphrase exist only on the failed machine.

That was my situation. One TrueNAS replica contained data I could not unlock because I no longer had the required encryption key. My Backblaze copy was recoverable only because I still had the Duplicati passphrase and could reconnect to the correct B2 backup location.

## The backup has dependencies

For my Immich library, the backup was not just a folder of pictures. A usable recovery involved the original media, Immich's database, the encryption settings, the B2 credentials, the backup passphrase and enough knowledge to rebuild the application.

If any one of those is missing, recovery becomes harder or impossible.

## Restore is the real test

The first time I truly tested the backups was after the incident. I restored the old Immich instance, decrypted files from Backblaze, copied data with resumable transfers, migrated into the active server and checked originals with SHA-256 hashes.

That process exposed missing items, duplicate assets, thumbnail errors and files that had not initially migrated. The backup job itself had never shown me that level of detail.

## What I believe now

A backup should answer five questions:

1. Where is the copy?
2. Is it complete?
3. Can I unlock it?
4. Can I restore the application and database?
5. How do I know the restored files are correct?

The goal is not to own more copies. The goal is to have at least one copy that can be recovered when the original is gone.

That is why the rest of my backup project is focused on documented restores, protected keys and regular verification—not simply creating more scheduled jobs.


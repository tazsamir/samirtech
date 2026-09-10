---
title: "Building a 3-2-1 Backup System for My Homelab"
date: 2026-09-12T09:00:00+01:00
draft: false
description: "The practical 3-2-1 backup plan I am building after recovering my photos from a wiped TrueNAS server."
tags: [backups, homelab, truenas, 321-backup]
series: ["Learning Backups the Hard Way"]
series_order: 3
---

After recovering my photos, I needed a backup design that did not depend on luck. The familiar 3-2-1 rule gives me a useful structure:

- 3 copies of important data;
- 2 different storage types or locations;
- 1 copy off-site.

It is not a magic formula, but it forces me to think about failure scenarios instead of trusting one NAS.

## My current layout

Normandy is my active TrueNAS server and runs the live Immich library. Its data is stored under the `Shepard` dataset, including the Immich data and PostgreSQL data.

Voyager is the second TrueNAS machine. It normally stays powered off or suspended, and receives local ZFS replication. That protects against some failures, but it is not enough on its own: a mistake, fire, theft or an encryption-key problem could affect both the data and the ability to recover it.

I also have removable storage used for protected keys and configuration archives. The long-term off-site plan is an encrypted copy at Backblaze B2 and an incremental copy to a Raspberry Pi 4 with attached storage at my parents' home.

## The three copies

The first copy is the live data on Normandy. The second is Voyager, which is a separate NAS and receives replicated data. The third will be off-site, using encrypted cloud storage and/or the Pi 4 copy.

The off-site copy matters because the two NAS machines are still in the same home. It protects against events that local replication cannot.

## Different storage and failure modes

Two TrueNAS systems provide useful redundancy, but they are still similar systems managed by the same person. That is why the design also includes cloud storage and removable/off-site media.

The copies should not all be online and writable at the same time. Voyager is normally off, and offline key storage gives me a recovery route if the online systems are compromised or accidentally changed.

## Keys are part of the backup

The recovery taught me that TrueNAS encryption keys and Duplicati/B2 passphrases must be backed up separately from the data. I have placed protected copies in a password manager and created encrypted archives on two USB drives.

The USB drives are not the main backup. They hold the information required to unlock and rebuild the real backups.

## The plan

My practical plan is:

1. Keep Normandy as the live service.
2. Replicate important datasets to Voyager on a schedule.
3. Keep Voyager powered off or suspended when it is not being used.
4. Back up Immich media, database dumps and configuration data to encrypted off-site storage.
5. Build the Pi 4 off-site copy and make it incremental over Wi-Fi.
6. Test restores and record the procedure.

The most important change is that every layer has a documented purpose. If one copy fails, I should know which copy to use next and what keys or software are required.


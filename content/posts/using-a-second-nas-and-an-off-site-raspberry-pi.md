---
title: "Using a Second NAS and an Off-Site Raspberry Pi"
date: 2026-09-15T09:00:00+01:00
draft: false
description: "Why my second NAS is useful but not enough, and how I plan to add an off-site Raspberry Pi backup."
tags: [backups, truenas, raspberry-pi, homelab]
series: ["Learning Backups the Hard Way"]
series_order: 6
---

A second NAS is one of the most useful upgrades I made after recovering my photos. It is also easy to overestimate what it protects against.

## What Voyager gives me

Voyager is my second TrueNAS server. It receives replicated data from Normandy and is normally powered off or suspended when it is not needed.

That gives me another local copy and a recovery system that can run the old Immich installation. It also means I can keep a copy away from the active server’s everyday changes.

## What it does not protect against

Both NAS systems are still in the same house. They can be affected by fire, theft, power problems, a network mistake or an administrator error. Replication can also copy unwanted changes if it is configured without useful snapshot retention.

The missing TrueNAS encryption key taught me another lesson: a replicated dataset is not useful if the recovery key is unavailable.

## Why the Pi 4 is useful

The next layer is a Raspberry Pi 4 with attached storage at my parents’ home. It will be an off-site destination for incremental backups from Normandy.

The Pi does not need to run the entire homelab. Its job is to hold a recoverable copy of selected important data, especially photos, videos, database dumps and configuration files.

Because it will be connected over Wi-Fi, the first copy may take time. After that, incremental transfers should only send changes. The backup will need careful scheduling so it does not overload the home connection or depend on the Pi being available every minute.

## Keeping it secure

The off-site Pi should not be treated as an open network share. It needs restricted access, encrypted transport, limited permissions and storage that is not writable by every service on the main server.

The data should also be encrypted if the attached drive could be removed or accessed independently. Keys must be stored separately and documented.

## The combined design

Normandy provides live services. Voyager provides a second local copy. The Pi provides distance. Backblaze B2 provides another off-site option for encrypted data and database backups.

Together, these layers reduce the chance that one mistake or one physical event removes every copy at once.


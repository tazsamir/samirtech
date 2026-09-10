---
title: "what-i-would-do-differently-if-i-started-again"
date: 2026-09-28T09:00:00+01:00
draft: false
series: ["Learning Backups the Hard Way"]
schema: 2
---

---
title: "What I Would Do Differently If I Started Again"
date: 2026-09-16T09:00:00+01:00
draft: true
description: "The practical lessons I took from wiping my TrueNAS server and recovering years of family photos."
tags: [backups, homelab, lessons-learned, truenas]
series: ["Learning Backups the Hard Way"]
series_order: 7
---

The most important lesson is that I should have slowed down before selecting the TrueNAS reset option while moving home. I did not read carefully enough, and the result was a wipe of the system hosting my photo library.

If I started again, I would change the following.

## I would separate data from experimentation

I would make the consequences of destructive operations obvious before pressing the button. I would export configuration, confirm snapshots and check the target dataset before resetting or rebuilding anything.

## I would document recovery while everything worked

I would record the TrueNAS dataset layout, Immich paths, database location, backup roots, B2 details and restore commands before an emergency. Recovery knowledge is hardest to reconstruct after a failure.

## I would protect keys first

I would keep TrueNAS encryption keys and Duplicati/B2 passphrases in a password manager, encrypted offline archives and a second safe location. The keys would be labelled clearly so I knew which system and dataset each one belonged to.

## I would test restores regularly

I would restore a sample of old photos and videos, rebuild a test database and check hashes on a schedule. I would not wait for a disaster to discover that an old backup was incomplete or depended on missing software.

## I would keep multiple recovery paths

The recovery worked because several imperfect sources overlapped: Voyager, Backblaze B2, Duplicati information, database dumps and older files. I would design those paths deliberately instead of discovering them after the loss.

## I would not delete old sources too quickly

After a migration, I would keep the source untouched until I had checked counts, sizes, old years, originals, videos, albums and database health. A migration summary is not the same as a verified restore.

## I would make the plan boring

The best backup system is not the cleverest one. It is the one that runs automatically, keeps more than one copy, stores its keys separately and has a restore procedure I can follow when tired and worried.

I was fortunate to recover my photos, including family memories going back to 2009. I do not want the next recovery to depend on fortune.


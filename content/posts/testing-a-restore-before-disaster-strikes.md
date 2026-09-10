---
title: "Testing a Restore Before Disaster Strikes"
date: 2026-09-14T09:00:00+01:00
draft: false
description: "The restore checks I am using after discovering that a backup job completing does not prove the data can be recovered."
tags: [backups, testing, data-recovery, homelab]
series: ["Learning Backups the Hard Way"]
series_order: 5
---

Before the TrueNAS reset, I had rarely tested a complete restore. The incident became the test, and it was much more complicated than I expected.

## What I test now

I do not only check that a scheduled job says “completed”. I test whether I can:

- locate the backup;
- authenticate to it;
- unlock encrypted data;
- restore a sample and a larger set;
- rebuild the database;
- open original photographs and videos;
- identify missing or duplicate files;
- explain the procedure to myself later.

## Start with a separate restore area

Recovered files first went to a separate TrueNAS location. I did not write directly into the live Immich library while I was still working out which backup root and user folder were correct.

That separation made retries safer and allowed me to compare the source and destination without confusing recovered data with live uploads.

## Check more than recent files

A restore can look successful because recent photos are present while older years are missing. I deliberately check different years, photos and videos, both users, and files from before the point where the original migration began.

In my case, some items from before 2024 did not appear in the first migration attempt. Reviewing the timeline exposed that gap.

## Hash checks and application checks

I use SHA-256 for selected file comparisons. A matching hash shows that the copied file is byte-for-byte identical to the file tested at the source.

That is only one part of the check. I also open the files, inspect the database, check albums and confirm that the active Immich app displays the results on the phone.

## Record the result

A restore test should produce notes: what was restored, from where, with which key, how long it took, what failed and how it was fixed. Those notes become the recovery guide for the next incident.

The best time to discover a missing key or broken restore command is during a planned test, not when the original server has already been wiped.


---
title: "My Post-Backup Verification Checklist"
date: 2026-09-24T00:00:00+01:00
draft: true
description: "A backup is not complete when the command exits: verify the snapshot, repository, monitoring and a representative restore."
tags: [backups, restic, homelab, recovery, monitoring]
---

A green backup job is useful evidence, but it is not the finish line. It may prove that a command exited successfully while leaving important questions unanswered: Was a new snapshot created? Did it contain the intended paths? Can the repository be read? Did monitoring record the result? Can a file actually be restored?

I use a short post-backup checklist to turn “the job ran” into evidence that the recovery path still works.

## 1. Read the job's own result

Start with the complete job output, not only the scheduler's exit code. I want to see:

- how many files and directories were processed;
- how much data was added;
- whether any source paths were skipped;
- warnings about unreadable files;
- the final snapshot identifier;
- the command's exit status.

An archive warning about a credential file or database is not harmless just because most other files were copied. If the missing file is required to restore the application, the backup is incomplete.

I keep logs bounded, but not so aggressively that the final summary or first useful error disappears.

## 2. Query the repository independently

Next I ask the repository what it contains. This is separate from trusting the output that claimed a snapshot was saved.

With Restic, that can be as simple as:

```sh
restic snapshots --latest 1
```

For repositories with several tags or source hosts, I filter by the exact stream I am checking:

```sh
restic snapshots --tag selected-data --latest 1
```

The newest snapshot should have the expected time, host label, paths and tags. A fresh snapshot for one tag does not prove that another tag is fresh. I track each backup stream separately because they often have different schedules and different failure modes.

## 3. Run an integrity check

A snapshot listing proves that metadata can be read. It does not prove that the repository's packs and trees are internally consistent.

My routine check is:

```sh
restic check
```

This validates the repository structure. Depending on repository size and available bandwidth, deeper data reads may belong on a less frequent schedule. The point is to choose the level deliberately rather than assuming that a successful upload validated everything.

Warnings need classification. For example, duplicate data left for a later prune is different from a damaged pack. I record the distinction instead of flattening every warning into either “fine” or “failed.”

## 4. Confirm the status record changed

My scheduled jobs write a machine-readable status record for monitoring. After a backup, I check that the record is:

- present;
- valid JSON;
- non-empty;
- newer than the run;
- associated with the correct backup tag;
- marked with the actual success or failure state.

A notification saying “success” is secondary evidence. The durable status record is what later freshness checks should evaluate.

When several backup types share one status file, I read the per-tag timestamp. A daily transfer cannot make a weekly source snapshot newer, and one successful tag must not hide another stale one.

## 5. Verify the notification path

If the job is meant to report through Telegram, email or another service, I verify that the recipient accepted a message and that the summary identifies the correct job.

This proves delivery for that run. It does not prove that a future missing job will be detected. Silence detection is a separate test: temporarily stop only the heartbeat or schedule, wait for the configured grace period, confirm the DOWN alert, restore the job, and confirm recovery.

That distinction matters. “The success message arrived” and “I will be warned if the job never starts” are different guarantees.

## 6. Restore something representative

A repository check is not a restore test. Periodically I restore a small representative sample into a new private directory rather than over live data.

For ordinary files, I verify more than the filename:

```sh
sha256sum original/example.jpg restored/example.jpg
```

For a picture, I also make sure it decodes. For application configuration, I inspect the restored tree and validate structured files, such as a Compose file, with the appropriate parser.

Those checks prove file recovery. They do not prove that a database can start or that a complete application has been recovered. I describe the evidence at the level actually tested.

## 7. Clean up temporary machinery

One-off recovery and migration work often leaves behind temporary timers, lock files, scratch restores or compatibility aliases. Once verification is complete, I remove only the temporary pieces and leave the durable schedule intact.

Then I read back the final scheduler configuration. Cleanup is part of the operation; forgotten temporary jobs can produce duplicate backups, lock contention or confusing notifications later.

## The checklist I keep beside the job

- [ ] The backup command completed without relevant skipped files.
- [ ] A new snapshot exists for the correct tag and source.
- [ ] The snapshot contains the expected paths.
- [ ] The repository integrity check passed or its warning was classified.
- [ ] The per-job status record is valid and fresh.
- [ ] The expected notification was delivered.
- [ ] A representative restore has succeeded on the agreed schedule.
- [ ] Temporary jobs and scratch data have been removed.
- [ ] The final schedule has been read back.

The value of this list is not complexity. It is that each line answers a different failure mode. A backup command protects data only when the repository, monitoring and recovery path agree that it worked.
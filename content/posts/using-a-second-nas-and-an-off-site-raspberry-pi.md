---
title: "Using a Second NAS and an Off-Site Raspberry Pi"
date: 2026-09-15T09:00:00+01:00
draft: false
description: "How my Raspberry Pi 4 backup uses encrypted Restic snapshots over Tailscale, and what a real picture and Docker-project restore proved."
tags: [backups, truenas, raspberry-pi, homelab, restic, recovery]
series: ["Learning Backups the Hard Way"]
series_order: 6
---

A second NAS is useful, but another copy in the same house is not protection against every disaster. My next backup layer uses a Raspberry Pi 4 with attached storage, intended for a separate location.

This is no longer just a plan. The initial selected-data backup has completed, repository checks have passed, and I have restored a picture and a Docker application's backed-up files into an isolated test folder. The important distinction: this verifies recovery from the Pi, not connectivity after relocating it or a complete replacement-server recovery.

## Where the Pi fits

The main NAS holds live data. A second NAS provides another local recovery layer. The Pi is an additional destination for selected important files and configuration snapshots, alongside the existing encrypted cloud-backup layer.

Two NAS systems in one house can still be affected by fire, theft, power problems or an administrator mistake. Replication can also propagate unwanted changes. Versioned backups and independent destinations serve different purposes; neither replaces checking that the data can actually be recovered.

The Pi is a backup target, not a replacement application server. It does not need to run the entire homelab.

## What is implemented

An always-on Linux Docker host orchestrates the job. It reads selected NAS data through read-only mounts and runs a digest-pinned Restic container. Restic encrypts and deduplicates the backup before sending it to the Pi over SFTP through Tailscale.

The flow is:

```text
Selected NAS files ───────────────┐
                                 ├─ Backup host → Restic → SFTP over Tailscale → Pi storage
Existing configuration snapshots ┘
```

The configuration snapshots are copied from an existing local Restic repository. The personal files are backed up directly as a separate snapshot group. Copying configuration snapshots does not create fresh database dumps or guarantee that every application's state is consistent.

The repository contains selected photo originals, a photo archive, important personal files, application configuration and existing database-backup files. It is not a backup of the entire NAS or every media collection. Regenerable thumbnails and transcoded video are not the priority.

## Tailscale DNS instead of a fixed address

The configured destination now uses the Pi's full Tailscale MagicDNS name rather than an IP address. DNS resolution and authenticated repository access were tested from the backup host, and the normal backup wrapper successfully listed snapshots after the change.

A name makes the configuration easier to understand, but it does not remove prerequisites: the recovery machine needs working Tailscale access, suitable access rules and DNS resolution. The Pi must also be online.

The actual tailnet name and addresses are deliberately omitted here. Changing the destination name must not be used as a reason to disable SSH host-key verification.

## Security and file permissions

There are separate layers:

- **Private connectivity:** Tailscale carries the connection without requiring a publicly exposed backup share.
- **SSH authentication:** a dedicated key authenticates the backup client.
- **Destination verification:** a pinned SSH host alias and strict known-host checking verify the server even when the connection address changes.
- **Encrypted backup contents:** Restic uses a repository password supplied through a private file, not a password embedded in a command.
- **Read-only sources:** the backup container reads the source data without write access through those mounts.

The backup process uses the authorized numeric identity and supplementary groups needed for the NAS permissions. I did not change host account IDs or broadly relax source ACLs to make the backup work.

Encryption is not immutability. A client with sufficient repository access can still damage its backup destination. The Pi therefore adds a layer; it does not make independent copies, restricted access or recovery-key management unnecessary.

## Scheduling and failure handling

The job is scheduled daily with an overall timeout. It follows a defined sequence:

1. Acquire a lock so scheduled and manual runs do not overlap.
2. Check free space on the destination and stop below a safety threshold.
3. Copy existing configuration snapshots while respecting the local repository's backup lock.
4. Back up the selected important files.
5. Run `restic check`.
6. Record the outcome and send a completion or failure notification.

There is no automatic offsite pruning in this setup. That avoids deleting recovery points during initial setup, but it also means retention and capacity need deliberate review. Deduplication reduces repeated storage; it does not make storage unlimited.

A completed upload, a structural repository check and a restore test are different evidence. The scheduled `restic check` is not a full read of every stored data block.

## The restore rehearsal that actually ran

I created a new private scratch directory on the backup host, separate from live application data. A recovery-only Restic container mounted the credentials read-only and the scratch directory as its output. It did not need the original NAS source mounts to retrieve the backup.

Two restores were performed:

- **A picture:** restored with Restic's `--verify`, independently compared by SHA-256 against a fresh repository extraction and the current original, then verified and fully decoded as a JPEG.
- **A Docker application's project folder:** Glance's backed-up Compose file, configuration, assets and environment file were restored. Every recovered regular file was independently compared against a fresh repository extraction. Compose validation passed without printing interpolated secrets.

No recovered application was started. Existing running containers retained their IDs, start timestamps and restart counts across the successful test. No live application directory or NAS file was overwritten.

That last point matters: blindly running a recovered Compose project could reuse production container names, ports, networks or bind mounts. An isolated file restore is a safe first test, not permission to start the recovered stack unchanged.

### A permissions issue worth documenting

The first attempts restored the data successfully, but inherited directory ownership and permissions blocked ordinary-user inspection. For the disposable inspection copies only, ownership and owner access were adjusted inside the scratch directory. Production permissions and NAS ACLs were not changed.

That adjustment is not a general restoration recipe. When recovering a real service, original ownership and ACL requirements must be preserved or deliberately mapped. A conveniently readable test copy should not be copied straight into production without review.

## A safe recovery sequence

These commands illustrate Restic's operations. They use placeholders, not my private repository configuration. Supply your repository, password file and verified SSH settings separately; do not paste passwords into commands.

```bash
# Inspect available recovery points and the intended contents.
restic snapshots
restic ls SNAPSHOT_ID

# Preview recovery into a new, empty scratch directory.
restic restore SNAPSHOT_ID \
  --include /path/in/snapshot \
  --target /separate/recovery-directory \
  --dry-run

# Recover and verify the selected files.
restic restore SNAPSHOT_ID \
  --include /path/in/snapshot \
  --target /separate/recovery-directory \
  --verify
```

Choose an explicit snapshot after checking its contents. A repository with configuration and personal-data snapshots can make an unqualified `latest` select the wrong kind of backup.

For a recovered Compose project, a useful next check is:

```bash
docker compose \
  --project-directory /separate/recovered-project \
  -f /separate/recovered-project/docker-compose.yml \
  config --quiet
```

This checks Compose configuration without starting the application. It does not establish that external mounts, databases, credentials or application behavior are correct.

Before a real replacement-server restore, check available disk space, recover credentials, verify source mounts where required, review each project's paths and permissions, and start one reviewed service at a time. Do not overwrite an entire system configuration tree or start all recovered projects at once.

## What remains unproven

The tests establish that the selected picture and Docker-project files can be retrieved and verified. They do **not** establish:

- a successful live replacement application;
- a full Immich database restore, including users and albums;
- a complete server or NAS recovery;
- a full repository data-read verification;
- reliable connectivity after moving the Pi to its intended remote location.

Photo originals recover the media, but Immich's organization also depends on a compatible database backup. An existing dump directory is not proof that a current, compatible dump has been restored successfully.

Finally, the repository password and required access material need independent secure storage. SSH access alone cannot decrypt a Restic repository. Keeping the only recovery secret on the machine being protected would undermine the whole design.

## The practical result

The Pi backup has moved from an idea to a working encrypted repository with a successful, non-destructive sample restore. The next confidence steps are a deliberately isolated application/database rehearsal and a connectivity check at the remote location—not simply collecting more successful-upload notifications.

Further reading: [Restic documentation](https://restic.readthedocs.io/en/stable/), [Tailscale MagicDNS](https://tailscale.com/kb/1081/magicdns), and [Docker Compose configuration validation](https://docs.docker.com/reference/cli/docker/compose/config/).

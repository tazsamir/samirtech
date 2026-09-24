---
title: "Restoring an Application from Backup"
date: 2026-09-17T09:00:00+01:00
draft: false
description: "What an isolated Navidrome database recovery rehearsal proved, how I kept it separate from production, and what remains untested."
tags: [backups, recovery, homelab, testing]
series: ["Learning Backups the Hard Way"]
series_order: 9
---

Getting files back is not the same as getting an application back. A database might be readable as a file while the application cannot open it. A container might start successfully while its users and library records are missing.

On **9 September 2026**, I went beyond restoring files: I started an isolated [Navidrome](https://www.navidrome.org/) instance using a recovered copy of its SQLite database. This article records that rehearsal, checked against its saved execution output on 17 September. It is not a new test performed today.

**The result was a successful application/database recovery rehearsal within a deliberately narrow scope.** The replacement served its HTML interface, and the selected database record counts were unchanged after shutdown. Login and playback remain the next test.

## What I was trying to prove

My [earlier restore article](/posts/testing-a-restore-before-disaster-strikes/) explains why I test backups. Here the narrower question was: **can the application start against its backed-up database without disturbing the live service?**

The source was a 6 September Restic configuration/application-data snapshot. The recorded restore recovered 8.065 GiB into a separate directory and verified 37,003 files. The next stage was to see whether Navidrome could actually use its recovered database.

From that restored tree, the rehearsal copied Navidrome's database and any accompanying SQLite journal files into a second, disposable working directory. It did not point the replacement at the live database.

SQLite is an embedded database, so there was no separate database-server container or SQL dump import. PostgreSQL applications, including Immich, need a different recovery procedure.

## Isolation came before startup

Starting the original Compose project unchanged would have been the wrong shortcut. Its ports, names, mounts and external integrations could collide with production.

The replacement instead used:

- A separate container name and a scratch database directory.
- The exact locally available image used by the live instance, with image pulling disabled.
- **No external networking and no published ports.**
- An empty, read-only music directory—not the live music library.
- A read-only container root filesystem, with a small writable temporary filesystem.
- No restart policy, dropped Linux capabilities and a no-new-privileges setting.
- Memory and CPU limits.
- Settings requesting disabled scanning, external services, plugins and playlist auto-import.

An empty music directory is useful for isolation but potentially dangerous to a library database if a scanner runs against it. That is why this was a disposable copy, not a substitute production service.

The saved report did **not** contain the expected scanner-disabled log message. The settings were supplied and the selected counts survived the short run, but the log did not independently confirm that safeguard.

## The commands, with private details removed

These are generalized excerpts of the executed rehearsal, not a universal recovery script. The paths below are deliberately supplied through variables; they must refer to inspected scratch directories. No commands in this article were rerun for publication.

The setup copied `navidrome.db` and any existing `-wal`, `-shm` or `-journal` companions from the restored backup tree. It counted records before startup, then launched the test container with this shape:

```bash
# Required: IMAGE is the exact locally available image ID used in the test.
# DATA is the disposable recovered database directory.
# EMPTY_MUSIC is a separate empty directory, not your real music library.
: "${IMAGE:?Set the verified local image ID}"
: "${DATA:?Set the absolute scratch database directory}"
: "${EMPTY_MUSIC:?Set the absolute empty scratch music directory}"

docker run -d --name navidrome-recovery-test \
  --pull=never --network none --restart no \
  --user "$(id -u):$(id -g)" \
  --read-only --tmpfs /tmp:rw,nosuid,nodev,size=64m \
  --cap-drop ALL --security-opt no-new-privileges \
  --memory 512m --cpus 1 \
  -v "$DATA:/data" -v "$EMPTY_MUSIC:/music:ro" \
  -e ND_SCANNER_ENABLED=false \
  -e ND_SCANNER_SCANONSTARTUP=false \
  -e ND_SCANNER_SCHEDULE=0 \
  -e ND_ENABLEEXTERNALSERVICES=false \
  -e ND_PLUGINS_ENABLED=false \
  -e ND_AUTOIMPORTPLAYLISTS=false \
  -e ND_DATAFOLDER=/data -e ND_MUSICFOLDER=/music \
  "$IMAGE"
```

These settings describe the image tested at the time. Before repeating the procedure with a different release, check its supported configuration and database compatibility. Do not combine a recovery rehearsal with an unplanned version upgrade.

Because no port was published, the HTTP check ran **inside** the container:

```bash
docker exec navidrome-recovery-test \
  wget -qO- http://127.0.0.1:4533/
```

The test retried during startup and required returned content containing an HTML marker. This confirmed that Navidrome reached its web interface; authenticated use and playback were outside the rehearsal.

The test then stopped the replacement before recounting database records:

```bash
docker stop --time 15 navidrome-recovery-test
```

For each selected table, the test ran a `SELECT count(*)` query against the scratch database. For example:

```sql
SELECT count(*) FROM "user";
SELECT count(*) FROM "media_file";
SELECT count(*) FROM "album";
SELECT count(*) FROM "artist";
SELECT count(*) FROM "playlist";
```

Finally it removed only the named replacement container. The scratch evidence remained separate from production. There was no promotion of recovered data into the live service.

## What the saved result actually showed

| Check | Recorded result |
| --- | --- |
| Replacement HTTP check | HTML returned successfully |
| User records | 1 before and after |
| Media-file records | 1,331 before and after |
| Album records | 71 before and after |
| Artist records | 242 before and after |
| Playlist records | 9 before and after |
| Production music mounted in the replacement | No |
| Published replacement ports | None |
| Replacement network mode | None |
| Live Navidrome start timestamp and restart count | Unchanged |
| Expected scanner-disabled log message detected | No |

The rehearsal command exited successfully. These are recorded results, not illustrative numbers.

Matching counts checked for obvious loss or an unintended rescan. Unchanged production start and restart values also confirmed that the rehearsal had not restarted the live container.

## What this rehearsal did not cover

I did not verify:

- Logging into the replacement with a recovered account.
- Browsing an authenticated library or opening individual playlists.
- Playing a recovered audio file, transcoding, or using a phone client.
- Restoring the music files as part of this application rehearsal.
- Recovery on a new host with no access to the original image cache.
- A production cutover, reverse proxy, DNS changes or remote-site access.
- Any PostgreSQL application or a whole-server rebuild.

The later Pi exercise was a separate file-level test: it recovered a picture and Glance project files, checked hashes and validated Compose syntax, but did not start a replacement application.

## A manageable next rehearsal

The next useful step would be a separately approved test with a small recovered music sample, isolated credentials and a deliberately controlled access path. Success would mean logging in, inspecting known records and playing that sample, while confirming production remained untouched.

Before doing that, I would document the exact backup, compatible image, ownership requirements, mount boundaries, network restrictions and cleanup steps. Private credentials and recovery locations belong in a protected runbook outside this website.

For the backup layers themselves, see [How My Restic Backup Protects the Homelab](/posts/how-my-restic-backup-protects-the-homelab/). The operational lesson here is smaller: **restore into isolation, start the application against the recovered copy, and state precisely what the checks prove.** A limited test with honest boundaries is more useful than calling a successful file copy a complete recovery.

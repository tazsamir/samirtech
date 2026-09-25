---
title: "Nextcloud Desktop Sync Exclusions and Diagnosis on Linux"
date: 2026-09-25
draft: false
description: "How to separate folder connections, exclude unsuitable data and diagnose a Nextcloud desktop sync that will not settle."
tags:
  - nextcloud
  - linux
  - file-sync
  - troubleshooting
---

Nextcloud Desktop normally works quietly, which makes a bad folder mapping easy to miss. A connection can report success while another connection handles the same remote path, or remain permanently busy because an active application keeps rewriting files beneath it.

My Linux setup has a broad Nextcloud connection and separate connections for Documents and Pictures. It also excludes game data and phone-import staging folders. The important lesson was that exclusions only make sense after each path has one clear owner.

## Know the client state

On my installation, the useful locations are:

```text
~/.config/Nextcloud/nextcloud.cfg
~/.local/share/Nextcloud/{folder}_sync.log
<sync-root>/.sync_*.db
sync-exclude.lst
```

Their roles differ:

- `nextcloud.cfg` describes accounts and folder connections;
- each `{folder}_sync.log` records activity for a connection;
- the hidden journal database stores the client's local sync state; and
- `sync-exclude.lst` tells a connection which paths to ignore.

Exact names and locations can vary by package and client version. Inspect the existing configuration rather than copying a path blindly. Configuration and logs may also contain private server names, usernames and filenames, so redact them before sharing.

## Map every connection first

Write each connection as a local-to-remote mapping:

| Connection | Local root | Remote root |
|---|---|---|
| Main | `~/Nextcloud` | `/` |
| Documents | `~/Documents` | `/Documents` |
| Pictures | `~/Pictures` | `/Pictures` |

Those local roots do not overlap, but the remote roots do. A main connection pointed at `/` also includes `/Documents` and `/Pictures` unless those directories are excluded from it.

A local overlap is more obvious:

```text
~/Nextcloud          <-> /
~/Nextcloud/Pictures <-> /Pictures
```

Both arrangements can give two journals responsibility for the same remote content. Symptoms include duplicated downloads, repeated uploads, conflicts, files returning after deletion and exclusions that appear inconsistent.

An exclusion applies to the connection that loads it. It does not automatically govern another connection.

## Give each path one owner

For a broad main connection plus dedicated Documents and Pictures connections, the main connection should exclude:

```text
Documents/
Pictures/
```

The narrower connections then own those remote subtrees. The alternative is one broad connection with no dedicated connections. Either can work; the unstable arrangement is allowing both designs to cover the same content.

For each path, answer two questions:

1. Which folder connection owns it?
2. Which exclusion file does that connection load?

## Exclude working data, not just large data

Some directories are poor candidates for live two-way sync even when their size is modest.

Game launchers and games may keep databases, caches and lock files open, while also using their own cloud synchronisation. Syncing those working files can create conflicts or preserve a state that was never internally consistent.

Phone imports have a similar problem. Import tools may rename, rotate, deduplicate or move photographs while Nextcloud is scanning them.

A calmer layout is:

```text
~/Pictures/Phone Import  # local staging, excluded
~/Pictures/Library       # completed collection, synchronised
```

Finish the import first, then move the settled files into the synchronised library.

If game saves need protection, identify the stable save files and back them up when the game is closed. Do not assume that syncing an active application directory is a safe backup.

## Exclusion syntax needs testing

Do not assume that `sync-exclude.lst` uses exactly the same rules as `.gitignore`, a shell glob or an `rsync` filter. Client versions can differ in their handling of path separators, wildcards, spaces and directory-only matches.

Start with a simple path relative to that connection's root:

```text
Games/
Phone Import/
```

Do not automatically add shell quoting or escaping:

```text
"Phone Import/"   # the quotes may become literal
Phone\ Import/     # this is not necessarily shell syntax
```

If the same directory name occurs in several places, test a path-specific rule supported by the installed client instead of using an over-broad name match.

After changing exclusions, restart the client or pause and resume the affected connection, then use a disposable file to verify the result. An exclusion normally prevents future synchronisation; it is not an instruction to delete a copy already present on the server.

## Read the matching log

When Pictures misbehaves, read the Pictures connection log, not only the main one:

```sh
less ~/.local/share/Nextcloud/Pictures_sync.log
```

Search for repeated evidence around the time of a test:

```sh
grep -Ei 'error|warning|excluded|ignored|conflict|permission|database|journal|locked' \
  ~/.local/share/Nextcloud/Pictures_sync.log
```

Useful clues include:

- a path matched by an exclusion;
- a missing or inaccessible local root;
- a server-side lock;
- an unsupported filename;
- a journal or transaction error;
- a missing remote path; or
- the same content discovered by two connections.

The client's activity view also distinguishes excluded, ignored, conflicted and errored items. Those states are not interchangeable.

## Treat the journal as state, not a cache to delete

The journal records what the client believes it has already seen. It helps distinguish a new file from a rename, deletion or conflict.

A damaged journal can cause repeated full scans, items that remain pending, transaction errors or changes that are never recognised. Deleting it should still be a last resort.

Rebuilding a journal forces the client to reconcile the local and remote trees again. That can surface old deletions and conflicts. Before attempting it:

1. stop the desktop client;
2. confirm the exact connection and journal path;
3. make sure important files have an independent backup;
4. preserve the original database by copying or renaming it; and
5. expect a complete rescan.

Do not open or modify the database while the client is running.

## A disciplined diagnosis

1. Pause every connection involved in an overlap.
2. Record all local and remote roots in a table.
3. Check for local nesting and remote nesting separately.
4. Confirm that each local root exists and has the expected ownership.
5. Identify the exclusion file used by each connection.
6. Check each rule relative to that connection's root.
7. Resume one connection at a time.
8. Create a disposable file in an included path and one in an excluded path.
9. Match the test time to the correct connection log.
10. Change one thing at a time.
11. Consider a journal rebuild only after mappings, permissions and exclusions have been ruled out.

A simple local-root check is:

```sh
for path in "$HOME/Nextcloud" "$HOME/Documents" "$HOME/Pictures"; do
    if [ -d "$path" ]; then
        printf 'OK:      %s\n' "$path"
    else
        printf 'MISSING: %s\n' "$path"
    fi
done
```

For exclusions, create obviously disposable test files and remove them after verification. Do not experiment with valuable documents or photographs.

## Common failure patterns

### An excluded directory still uploads

The rule may belong to the wrong connection, another connection may cover the same path, the pattern may be relative to a different root, or the client may not have reloaded it. Content uploaded before the rule was added can also remain on the server.

### Documents or Pictures appear twice

A main connection probably owns `/` while a dedicated connection also owns `/Documents` or `/Pictures`. Exclude the subtree from the main connection or simplify the setup to one connection.

### A phone import never settles

The import process and Nextcloud are both changing the directory. Use a non-synchronised staging area and move completed files afterwards.

### Game files repeatedly conflict

Exclude active caches, lock files and databases. Protect stable saves with a suitable backup rather than syncing a live application state.

### Logs report database errors

Stop the client, then check storage space, filesystem health and permissions before touching the journal. Preserve the original database if a rebuild becomes necessary.

## Sync is not backup

A two-way sync can propagate accidental deletion, unwanted edits, corruption and ransomware-encrypted files. Server-side versioning and deleted-file retention can help, but their usefulness depends on configured limits and available space.

A backup should be stored independently, retain versions, resist ordinary sync operations and be tested by restoring files.

Use Nextcloud to make working files available on several devices. Use a backup to make them recoverable after the synchronised copy has gone wrong.

## A stable target layout

```text
Main connection
  Local:  ~/Nextcloud
  Remote: /
  Excludes:
    Documents/
    Pictures/
    Games/

Documents connection
  Local:  ~/Documents
  Remote: /Documents
  Excludes:
    application caches and temporary data

Pictures connection
  Local:  ~/Pictures
  Remote: /Pictures
  Excludes:
    Phone Import/
    Camera Imports/
```

The exact paths are less important than the rule behind them: every synchronised path should belong to one folder connection, and every exclusion should be attached to the connection that actually sees it.

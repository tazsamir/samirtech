---
title: "Docker, NFS and TrueNAS: Getting Permissions and ACLs Right"
date: 2026-09-25
draft: false
description: "How I handled different numeric identities across Docker and TrueNAS without renumbering established users."
tags:
  - docker
  - nfs
  - truenas
  - permissions
  - acls
---

An NFS share can mount successfully, show every file and still refuse to let a container create, rename or delete anything.

The confusing part is that the failure crosses three systems:

```text
container process
  -> bind mount on the Docker host
  -> NFS client identity
  -> TrueNAS share mapping
  -> dataset ACL
```

In my case, the ordinary account on the Docker host used UID and GID `1000:1000`, while the established TrueNAS account used `3000:3000`. The names could match, but those numbers did not.

The safe solution was not to renumber either machine. It was to make the server-side identity handling and ACL explicit, then test the effective access through the same path the application uses.

## Names are for people; IDs cross the wire

Traditional Unix permission checks are based primarily on numeric user and group IDs. These two identities are not automatically equivalent:

```text
Docker host: media -> 1000:1000
TrueNAS:     media -> 3000:3000
```

NFSv4 can support name-based identity mapping in a deliberately configured environment, but using NFSv4 does not prove that mapping is working. The useful question is not what the account is called. It is which identity TrueNAS sees for the request.

## Why I did not change every UID

Renumbering an established account can affect far more than one share:

- existing dataset ownership;
- local bind mounts and container configuration;
- SMB access;
- application databases;
- backups and replication jobs;
- snapshots containing old ownership; and
- other NFS clients.

A recursive `chown` is especially risky on shared data. It can destroy intentional ownership boundaries or interact badly with inherited ACL entries.

Changing IDs may be appropriate during a planned migration, but it is a poor first response to one permission error.

## Let TrueNAS define the access policy

TrueNAS is authoritative for the exported dataset. Depending on the design, the right solution may be:

- an ACL entry for the identity arriving from the Docker host;
- access through a shared group;
- a narrowly scoped NFS user or group mapping;
- deliberately configured NFSv4 identity mapping; or
- a dedicated dataset for one application's data.

Map-root and map-all style settings have very different consequences. Root mapping controls how remote root is represented. Mapping all requests to one account can suit a tightly scoped single-purpose export, but it also removes useful attribution and can grant more access than intended.

Use the narrowest option that works, on the smallest practical export. Identity mapping does not replace the dataset ACL: the resulting server-side identity must still have the required permissions.

## Container identity is a separate decision

Some images accept `PUID` and `PGID`:

```yaml
services:
  application:
    image: <IMAGE>
    environment:
      PUID: "1000"
      PGID: "1000"
    volumes:
      - /mnt/<share>/<directory>:/data
```

These variables are image conventions, not Docker features. Other images use Compose's `user` setting, and some start as one account before dropping privileges to another.

Verify the running process rather than trusting the Compose file:

```sh
docker exec <container> id
docker exec <container> ps -eo user,group,pid,args
```

Changing a container to `3000:3000` might make one NFS write succeed while breaking its local configuration directories. Server-side mapping or ACLs are often less disruptive.

## Root in a container is not necessarily root on the NAS

NFS commonly applies root squash. A request from client UID `0` becomes an anonymous or restricted identity on the server.

That is a security boundary, not an inconvenience to disable casually. A root container may be able to read or create files yet fail to change ownership. First establish whether the application truly needs `chown`, or merely needs create, write, rename and delete access.

If elevated mapping is genuinely required, keep it limited to a dedicated export and trusted clients.

## POSIX mode bits are not the whole ACL

Commands such as these remain useful:

```sh
ls -ldn /mnt/<share>/<directory>
stat -c '%u:%g %a %n' /mnt/<share>/<directory>
```

They show numeric ownership and POSIX mode bits. They do not necessarily show the complete policy on a dataset using NFSv4 ACLs.

An NFSv4 ACL can contain named users and groups, allow and deny entries, inheritance flags, and distinct rights for creating, deleting and traversing. It should not be treated as a verbose version of `chmod`.

Inspect and modify the dataset with tools appropriate to its configured ACL type. Do not use recursive `chmod`, `chown` or ACL replacement as a substitute for understanding the existing policy.

## Inheritance matters

Fixing access to the top-level directory is not enough. Newly created files and subdirectories need suitable inherited entries.

Test that the intended identity can:

- traverse each parent directory;
- create a file and a directory;
- append to and rename the file;
- delete it from the directory; and
- create content another intended service can subsequently read or modify.

A parent ACL can permit creation while producing a child that another service cannot process. Existing children also do not necessarily inherit a newly added rule retrospectively.

## Test the direct path

A successful write in the TrueNAS shell proves little about a request made by a container. Test each layer.

### On the Docker host

```sh
findmnt -T /mnt/<share>/<directory>
findmnt -no SOURCE,FSTYPE,OPTIONS -T /mnt/<share>/<directory>
ls -ldn /mnt/<share>/<directory>
```

Then perform a harmless test as the relevant host account in a disposable directory:

```sh
sudo -u <host-user> sh -c '
  f="/mnt/<share>/<test-directory>/.permission-test-$$"
  : > "$f" && printf "test\n" >> "$f" &&
  mv "$f" "$f.renamed" && rm -- "$f.renamed"
'
```

### Inside the container

```sh
docker exec <container> id

docker exec <container> sh -c '
  d="<container-test-path>"
  f="$d/.permission-test-$$"
  : > "$f" && printf "test\n" >> "$f" &&
  mv "$f" "$f.renamed" && rm -- "$f.renamed"
'
```

This is the decisive test because it uses the effective process identity, bind mount and NFS route used by the application.

Where possible, also reproduce the application's real operations. Creating a file and renaming one can require different ACL rights.

## A safe diagnostic sequence

1. Record the host user's numeric identity with `id <host-user>`.
2. Record the container's effective UID, GID and supplementary groups.
3. Confirm that the expected NFS mount backs the exact host path.
4. Confirm that neither the NFS nor container mount is read-only.
5. Identify the dataset's ACL type.
6. Review the complete dataset ACL and inheritance flags.
7. Review map-root, map-all and anonymous identity settings.
8. Determine which identity TrueNAS sees after mapping.
9. Test create, append, rename, delete, `mkdir` and `rmdir` on the host.
10. Repeat the test inside the container.
11. Inspect the owner and ACL of newly created objects.
12. Change one layer at a time and repeat the same tests.

Before changing a share or ACL, preserve its configuration. Test against a non-production directory and avoid broad recursive operations.

## The resulting design

For this kind of environment, a maintainable design is:

1. keep the Docker host's established `1000:1000` identity;
2. keep the TrueNAS user's established `3000:3000` identity;
3. run each container with a deliberate, documented account;
4. translate or authorise that access on the TrueNAS side;
5. preserve root squash;
6. apply and verify suitable inheritance; and
7. test access through the actual container path.

The aim is not to make every number identical. It is to make the server's interpretation of each request explicit, narrow and testable.

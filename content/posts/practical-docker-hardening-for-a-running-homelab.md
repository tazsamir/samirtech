---
title: "Practical Docker Hardening for a Homelab That Is Already Running"
date: 2026-09-24T00:00:00+01:00
draft: true
description: "An incremental way to improve logging, restart behaviour, health checks, exposure and backups without rebuilding a working Docker host."
tags: [docker, homelab, security, monitoring, backups]
---

Most Docker hardening guides begin with an empty server. Mine was already running useful services, which changes the problem.

On a live homelab, the goal is not to apply every possible security control in one maintenance window. It is to reduce known failure modes without breaking dependencies, losing application state or turning a working server into an experiment.

My approach is incremental: inventory first, make one bounded class of change, recreate only affected services, and verify application behaviour rather than stopping at `docker ps`.

## Start with declared and effective state

Compose files show intent. Docker's live state shows what actually happened. I inspect both without dumping resolved environment variables or secret-bearing configuration.

For each container I care about:

- image and Compose project;
- state, health, restart count and OOM status;
- restart policy;
- logging driver and limits;
- CPU and memory limits, if any;
- published ports and network mode;
- mounts and whether their sources exist;
- privileged mode, capabilities and devices;
- health-check definition;
- the authoritative Compose file path.

A running container is not the same as a working application. I also probe the service's real local endpoint and its important dependencies.

## Bound logs before they become a disk incident

Unbounded JSON logs can consume the host filesystem quietly. Compose makes the limit explicit:

```yaml
services:
  example:
    image: example/image:1.2.3
    logging:
      driver: json-file
      options:
        max-size: "10m"
        max-file: "3"
```

This is not a substitute for centralized logs where I need history. It is a safety boundary on the local copy.

Changing the Compose file does not alter an existing container. I validate the file, recreate only the affected service, then inspect the live logging configuration. I avoid broad `docker compose up -d` commands across unrelated projects.

## Choose restart policies by role

`restart: unless-stopped` is a sensible default for long-running services that should return after a crash or host reboot:

```yaml
restart: unless-stopped
```

It is not correct for everything. One-shot jobs should finish and stay finished. Maintenance containers may need an external scheduler. A crash-looping service should not be treated as healthy merely because Docker keeps restarting it.

After changing a restart policy, I read the effective policy from Docker and make sure an intentionally stopped service remains intentional.

## Add health checks that prove something

A process-level check often proves only that a binary exists. A useful health check exercises the narrowest endpoint that represents readiness.

For an HTTP application:

```yaml
healthcheck:
  test: ["CMD", "wget", "-q", "--spider", "http://127.0.0.1:8080/health"]
  interval: 30s
  timeout: 5s
  retries: 3
  start_period: 30s
```

The exact path matters. Some services correctly return `404` at `/` while exposing dedicated health and readiness endpoints elsewhere.

Health checks also have limits. A web interface returning `200` may still have a broken database, inaccessible storage or a failed background scheduler. I use dependency-aware checks where the application provides them, and keep external monitoring for user-visible availability.

## Be cautious with memory limits

Tight memory limits look like hardening but can create outages when chosen from a single quiet-period measurement. Databases, media processing and indexing workloads are bursty.

I first collect normal and peak behaviour, check prior OOM events, and classify the service. Then I add a limit only when I understand the consequence of hitting it. A monitoring alert on unusual growth is often a safer first step than an arbitrary cap.

The goal is to prevent one service from consuming the host, not to make every graph look tidy.

## Reduce exposure without breaking access

For every published port I ask who actually needs it:

- the local machine only;
- the LAN;
- a private overlay network;
- a reverse proxy;
- the public internet;
- nobody, because only another Compose service uses it.

An internal dependency normally needs a Docker network, not a host port. A management interface may be bound to a specific private address rather than every interface.

I inventory all real access paths before tightening bindings. Media discovery, guest-network routing, reverse proxies and overlay access can depend on different interfaces. A successful request from the Docker host does not prove that the intended client can connect.

I also treat a read-write Docker socket mount as host-equivalent privilege. Services that only need selected Docker metadata should use a constrained socket proxy where practical.

## Protect state before recreating containers

Before changing a stateful service I identify:

- bind mounts and named volumes;
- its database type;
- whether a live filesystem copy is consistent;
- the existing backup and restore method;
- any dependency order needed during shutdown and startup.

A folder full of copied database files is not automatically a usable database backup. Native dumps or an application-consistent quiesced copy are safer, and the restore procedure should be tested away from live data.

I make a narrow backup before the change, validate the Compose file, and keep the previous configuration available for rollback.

## Recreate narrowly, then verify broadly

My change loop is deliberately repetitive:

1. Back up the affected configuration and state.
2. Edit the authoritative Compose file.
3. Parse and validate it without printing secrets.
4. Recreate only the affected service and necessary dependencies.
5. Wait for any startup period.
6. Check container state, health, restarts and OOM status.
7. Exercise the actual application endpoint.
8. Test its critical dependency or scheduled function.
9. Inspect fresh bounded logs for new errors.
10. Read back the effective Docker policy.

A bind-mounted configuration deserves special attention. Some editors replace a file atomically, leaving a running container attached to the old inode. If the host and in-container files differ, a reload may say “unchanged” because the container cannot see the replacement. After validating the new file, I recreate only that service so the mount binds to the current inode.

## The order that gave me the most value

For an existing host, I prioritize changes like this:

1. Repair confirmed broken dependencies and scheduled jobs.
2. Verify backups and a representative restore.
3. Bound local logs.
4. Fix unsuitable restart policies.
5. Add meaningful health checks.
6. Reduce unnecessary port and Docker-socket exposure.
7. Add resource limits only from measured evidence.
8. Move floating image updates toward a deliberate review process.
9. Prune only objects proven unused.

This is slower than rewriting the stack around an ideal template. It is also safer. Hardening a live homelab is operations work: preserve what already works, make the smallest justified change, and prove that the application—not merely the container—still does its job.
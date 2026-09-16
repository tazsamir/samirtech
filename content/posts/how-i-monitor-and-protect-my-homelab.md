---
title: "Is My Homelab Actually Working? Glance, Gatus, Grafana and Useful Alerts"
date: 2026-09-16T10:00:00+01:00
lastmod: 2026-09-16T12:45:00+01:00
draft: false
description: "How I use Glance, Gatus, Grafana and Gotify to see what is available, investigate problems and keep security notifications useful."
tags: [homelab, monitoring, security, self-hosting]
---

Installing an application is the easy part. Knowing whether it still works a week later is a different job.

Once I had several services running, opening each one to check it became tedious. A photo library might load its homepage but fail to access storage. A container might be running while its application is stuck. Several perfectly healthy apps might appear offline because the shared DNS server or reverse proxy has failed.

I wanted a simple way to answer three questions: **what can I use, what has gone wrong, and does it need my attention now?**

My setup uses Glance, Gatus and Grafana for different parts of that job, with Gotify for local monitoring notifications and an external Healthchecks.io heartbeat that alerts me directly through Telegram. Security monitoring sits alongside availability monitoring, rather than being confused with it.

## The short version

| Tool | What I use it for | What it does not prove |
| --- | --- | --- |
| **Glance** | My everyday starting page: app links and quick availability indicators | That every feature behind a working page is healthy |
| **Gatus** | Repeated endpoint checks, response times and a record of availability | That a successful HTTP request means a complete application workflow works |
| **Grafana** | Looking at metrics and logs to understand a problem or a trend | That a graph without errors guarantees a secure system |
| **Gotify** | Bringing selected notifications to my attention | That every failure has an alert rule, or that a delivered message was read |
| **Healthchecks.io → Telegram** | Warning me when the server stops reporting to an independent service | Which dependency failed, or whether individual applications still work |

The distinction matters. A dashboard is somewhere I look. A notification is something that comes to me. Neither is useful unless the underlying checks mean something.

## Glance: the page I open first

[Glance](https://github.com/glanceapp/glance) is the front door to the lab. It gives me a manageable set of bookmarks and monitor widgets instead of a collection of addresses and ports to remember.

My configuration groups checks by the services on the Docker host and the NAS web interfaces. That makes it easier to see whether one app needs attention or whether several services on the same machine have stopped responding.

It is particularly useful from a phone: open one page, find the service, then follow its link. I do not need a full metrics dashboard just to open the music library.

But I treat a green indicator as a narrow statement: **the configured check succeeded**. It is not a promise that login, playback, uploads, background jobs and storage access all work. A bookmark alone is not a health check either.

## Gatus: checking rather than guessing

[Gatus](https://github.com/TwiN/gatus) performs recurring checks and keeps the results. This is where I look when I want more than a quick current-state indicator.

The checks in my setup include application web endpoints, the HTTPS route to Grafana, and DNS checks for both local names and upstream resolution. They test expected response conditions and response-time limits.

That helps separate failures which otherwise look identical in a browser:

- **One application fails:** start with that service and its dependencies.
- **Several friendly URLs fail together:** investigate shared DNS and proxy dependencies before restarting every app.
- **A direct application route works but its friendly URL does not:** the access path may be broken even though the application is running.
- **A page responds but is consistently slow:** there may be a developing problem rather than a complete outage.

Where the check runs matters too. A server-side check cannot prove that a phone using different DNS settings, another network or a VPN will get the same result. When a device has a problem, I still test from that device.

**An important detail in my current setup:** Gatus is providing status checks, but its configuration does not currently define alert providers. A failed Gatus check therefore should not be described as automatically producing a Gotify notification. That is a separate integration to configure and test if I want it.

## Grafana: understanding why

[Grafana](https://grafana.com/oss/grafana/) is where I go for more detail. Glance is the starting page; Gatus records whether checks pass; Grafana helps me investigate what was happening around a failure.

Behind Grafana, Prometheus collects metrics and Loki stores logs. Exporters and collectors make host, container and other infrastructure information available to those systems. Grafana displays the data; it does not create reliable measurements simply because a dashboard exists.

My provisioned dashboards cover areas such as server health, the Docker fleet, storage, NAS applications, internet connection health and container logs.

The useful questions are practical:

- Was memory pressure building before the application stopped responding?
- Is the disk getting full, or is storage activity unusually high?
- Did the slowdown affect the whole host or just one container?
- Did errors appear around the time of an update?
- Is an internet problem affecting several otherwise unrelated services?

A graph can show a trend that a green status dot misses. A service can still respond while its remaining disk space is steadily disappearing.

I also need to distinguish **no data** from **everything is fine**. If an exporter stops reporting, or a dashboard query no longer matches the available metrics, an empty panel is not evidence of good health. Check the data source and timestamps before trusting the picture.

## Notifications: useful enough that I keep reading them

[Gotify](https://gotify.net/) is the notification hub in this setup. It gives separate monitoring jobs somewhere to send a short, actionable message rather than requiring me to watch dashboards all day.

The surrounding setup includes container-health notifications, image-update monitoring and a dedicated security-notification collector. These are separate paths; their coverage is not identical to the list of checks shown in Gatus.

An image-update notice means a new image is available for review. It does not mean the running version is compromised, and it is not permission to update every stateful service automatically. I want to know what changed, check any migration requirements, and have a recovery path first.

For security notifications, the design is deliberately low-noise: grouped CrowdSec decisions, repeated SSH failures rather than every individual failed attempt, and selected higher-severity Falco events. Filtering is a trade-off, not complete coverage. Lower-severity events may still deserve investigation even when they do not generate an immediate notification.

A useful alert should tell me:

1. Which service or security component needs attention.
2. What was observed, without overstating what it proves.
3. When it happened and where I can investigate.
4. Whether the condition has recovered, when recovery reporting is supported.

Repeated copies of the same message do not make the system safer. Grouping, sensible thresholds and cooldowns help keep notifications readable. Failed delivery needs retry handling, and a labelled test message is how I check the delivery path—not an assumption that a configured URL must work.

## Security is a separate layer

Availability monitoring asks, “Does this respond?” Security monitoring asks, “Is something happening that should not be?” A service can answer every health check while being misconfigured or compromised.

My broader approach combines restricted access, SSH keys, protected credentials, deliberate updates and recoverable backups with monitoring. The local HTTPS arrangement is explained in [Local HTTPS Without Exposing My Homelab to the Internet](/posts/local-https-without-exposing-my-homelab/). A certificate encrypts and authenticates a connection; it does not decide who is allowed to reach the service.

Two security tools in the setup have distinct roles:

- **[CrowdSec](https://www.crowdsec.net/)** analyses supported activity and can create decisions about suspicious sources. A decision is not proof that traffic was blocked: enforcement depends on a correctly configured and functioning remediation component, often called a bouncer.
- **[Falco](https://falco.org/)** detects runtime behaviour against rules. An alert is a reason to investigate the event, application and timing—not automatic proof of malware, and not something to dismiss just because the container looks healthy.

I do not want raw command arguments, tokens or sensitive log contents copied into phone notifications. The notification should identify the problem; detailed evidence can stay in the protected system where it belongs.

Dashboards need protection too. They can reveal application names, infrastructure layout, activity and logs. A monitoring page is not harmless just because it does not contain an obvious “delete” button.

## What I do when something looks wrong

For a photo-library problem, my investigation would look like this:

1. **Start in Glance.** Is it one service or a wider group? Follow the link and confirm what actually fails.
2. **Check Gatus.** Did the endpoint fail, become slow, or remain reachable? Is DNS affected too?
3. **Use Grafana.** Compare the relevant time window with resource metrics, storage signals and logs. Check that the data is fresh.
4. **Read related notifications.** Look for a matching health event, security event or recent maintenance notice. Timing is evidence to investigate, not proof of a cause.
5. **Make the smallest justified change.** Avoid restarting the entire stack when the evidence points to one dependency.
6. **Verify the actual user task.** Load a photo, play a file or complete the action that originally failed. A recovered status dot alone is not enough.

This is an example troubleshooting method, not a claim that a particular outage occurred or that every application has a synthetic end-to-end test.

## The monitoring system can fail as well

There is an obvious limitation to running much of this on the same always-on host: if that host goes down, its dashboards and notification jobs may go down with it.

No message does not necessarily mean no problem. It can mean the collector stopped, the network failed, or Gotify became unreachable.

### What I added: an external heartbeat

I have now added a heartbeat to the hosted [Healthchecks.io](https://healthchecks.io/) service. A scheduled job on the always-on server sends an outbound HTTPS request every five minutes. If those requests stop arriving, Healthchecks can alert me directly through Telegram, without relying on the server or its Gotify instance to send the message.

```text
Home server → outbound heartbeat → Healthchecks.io
                                       │
                              Missing or failed heartbeat
                                       │
                                       v
                                    Telegram
```

This is sometimes called a *dead man’s switch*: instead of trusting silence, an independent system expects regular evidence that the job is still running. It requires no new inbound port, public dashboard or second local monitoring stack.

The installation preserved the existing scheduled jobs. The ping URL is kept in an owner-only configuration file outside the website repository, with bounded connection and request timeouts and retries. A private success timestamp is updated only after an accepted ping. The URL is a secret: anyone who knows it could send false heartbeats, so it does not belong in an article, screenshot or public configuration example.

### Timing and what it actually tells me

The server's verified sending interval is **five minutes**. The timing I settled on for the external check is a **five-minute period with a thirty-minute grace window**. With those settings saved in Healthchecks, an alert is due about **thirty-five minutes after the last successful ping**. The grace window avoids alerts for short interruptions, at the cost of slower warning.

Those provider-side settings still need confirmation in the account dashboard; the ping URL can report success or failure but cannot read or change the check's schedule. The server sending every five minutes does not, by itself, establish the alert deadline.

A missing heartbeat can mean the server is down, its scheduler has failed, or the home internet or power connection is unavailable. It does not distinguish those causes. Telegram delivery also depends on Healthchecks, Telegram and my device having connectivity; during a home internet outage, a phone may need mobile data.

This is a **host-reporting check, not an application-health check**. It can keep succeeding while Docker or an individual app is broken. A separate heartbeat that reports success only after selected local health checks pass would be a useful next layer, but it has not been installed.

### The test I actually completed

On 16 September 2026, the first ping was accepted and a subsequent scheduled run was independently observed. I then tested the notification path by sending an explicit failure signal to Healthchecks, followed by a successful recovery heartbeat. Healthchecks acknowledged the failure request, accepted the recovery, and I received both the **DOWN** and **UP** messages in Telegram.

No server or application was stopped, and the normal five-minute schedule remained in place. This confirms the explicit failure-and-recovery notification path. It does **not** yet prove the missing-heartbeat timeout: that needs a separate controlled pause of only the heartbeat job, leaving the Healthchecks check enabled, followed by restoring the job and confirming recovery.

The external heartbeat reduces the silent-host-failure gap; it does not make the local services redundant. The same applies to DNS: monitoring one resolver does not remove it as a single point of failure.

Monitoring configuration, dashboard definitions and alert rules also belong in the backup plan. Recovery instructions should remain accessible without relying entirely on the system being recovered.

## What was checked for this draft

A read-only check on 16 September 2026 found Glance, Gatus, Grafana, Prometheus, Loki and Gotify running. Gatus's returned latest results were successful for its configured endpoints, and Grafana's health endpoint reported its database as OK. The configured checks, Glance widgets and dashboard definitions were also inspected.

CrowdSec and Falco reported active. The dedicated security collector's timer was enabled, and its most recent recorded run had a successful exit status. Those are operational checks, not proof that every security event is collected or that every notification reaches a device.

The later Healthchecks failure-and-recovery test confirmed Telegram delivery as described above. It did not test the Gotify or security-alert delivery paths. This drafting work did not trigger a security event, verify CrowdSec enforcement, test every Grafana panel or audit the complete network boundary. Privileged SSH-policy inspection was unavailable, so this is not a fresh certification of the host's effective SSH settings. The article describes the setup and its limits, not a security guarantee.

## Why I think this matters

The point is not to build an enterprise monitoring department at home. It is to stop discovering problems only when someone wants to use an application.

Good monitoring gives me an earlier warning, a clearer place to start and less temptation to change things blindly. Useful notifications let me step away without pretending that I am watching every graph. Security checks help flag behaviour that an uptime check cannot see.

Backups complete the picture, but they answer a different question: **can I recover?** A green dashboard cannot answer that. That takes [a restore test](/posts/testing-a-restore-before-disaster-strikes/).

My rule is simple: **Glance to get there, Gatus to check availability, Grafana to investigate, and notifications for the things worth interrupting me about.** Keep the checks honest, understand the gaps, and test the user-facing result.

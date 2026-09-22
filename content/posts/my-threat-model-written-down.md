---
title: "My Threat Model, Written Down"
date: 2026-09-22T00:00:00+01:00
draft: true
description: "What I am actually defending against by self-hosting, what I accept instead, and why a written threat model stops scope creep in the lab."
tags: [security, self-hosting, homelab, threat-model]
---

Most homelab security writing skips the first question and goes straight to tools: a firewall, a VLAN, a fail2ban rule, a hardened kernel. But every one of those defends against *something specific*, and if you never say what, you cannot tell whether the tool is doing its job or just making you feel careful.

This is my answer to the question I rarely see asked: **defend against what?**

## The threats I actually face

A threat model is a list of what could go wrong, ranked by how much I care and how likely it is. Mine is not a cybersecurity-specimen list; it is boring and specific.

1. **A platform decides my data stops being mine.** A streaming service delists the thing I bought; a photo service changes its terms; a notes app sunsets. This is the threat that started the whole lab, and it is the one self-hosting genuinely answers.

2. **A hard drive or NAS dies.** Storage is the thing I most expect to fail, because storage always fails eventually. Backups are the answer, and they are the one area where I test rather than assume.

3. **I break something myself.** A bad command, a deleted dataset, an update that breaks a service. I am my own most likely cause of data loss, ahead of any attacker.

4. **A service I expose is compromised.** Anything reachable beyond my home network is a target, however small. The answer here is to expose as little as possible and to isolate what must be exposed.

5. **Theft, fire, or a serious house problem.** Not exotic; it is why an off-site copy matters and why "another machine in the same house" is not a backup.

Unranked because I mostly accept them: targeted attacks by a determined adversary with resources and patience. A person who specifically wants *my* data, and will spend months on it, is not a realistic threat for a homelab, and protecting against that would cost far more than the data is worth. Knowing that I am *not* defending against it is what keeps the rest of the model sane.

## What I accept instead of defending

A threat model is also a list of things I deliberately do not try to solve, because the cost would outweigh the value.

- **I do not run my own email.** Delivery, reputation, filtering, spam, security and recovery are operational work I will not take on. I accept a hosted provider here.
- **I do not chase perfect isolation.** A guest VLAN, scoped DNS and a reverse proxy that refuses non-local clients cover the realistic exposure. I do not need an enterprise zero-trust mesh at home.
- **I accept that convenience has a floor.** Some services are used from devices where I cannot enforce every setting, and I accept that boundary rather than pretending to control it.

These acceptances are not failures. They are the part of the model that stops a homelab from becoming a second job.

## Why writing it down matters

The moment I stop naming the threats, I start defending against *everything*, which means defending against *nothing specific*. A written list gives every security decision a standard to be measured against:

- A new container that opens a port only makes sense if it answers one of the threats above.
- A new backup destination only earns its place if it protects a *named* piece of data from a *named* failure.
- A hardening tip I read online is only worth adopting if it moves the needle on something on this list.

The list also keeps me honest about the gaps. The Pi is not yet confirmed off-site, so threat five is partially unhandled — and I would rather have that written down than have a tidy-looking setup that silently pretends otherwise.

## The one-sentence version

Defend against losing my data and against casual exposure. Test the parts that fail reliably. Refuse to spend effort on threats that are not actually mine, and write the whole list down so the next security decision has somewhere to aim.
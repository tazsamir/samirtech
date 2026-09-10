---
title: "Building a More Useful Homelab with Hermes Agent"
date: 2026-09-10T17:00:00+01:00
draft: false
description: "How I have been using Hermes Agent to document, maintain and improve my Linux workstation and homelab."
tags: [hermes-agent, linux, homelab, automation]
---

# Working with Hermes Agent

I started experimenting with Hermes Agent in September 2026. It is an AI agent that runs in the terminal and can use tools to inspect files, research documentation, operate on systems and help turn a vague idea into a tested result.

The interesting part is not simply asking an AI questions. Hermes can work alongside the systems I already use: Fedora and KDE on the desktop, Docker on the N100, TrueNAS for storage and Telegram for remote status updates. The useful work happens when the agent checks the real system instead of guessing.

## What we have been doing

- Auditing and maintaining the homelab without removing services destructively.
- Improving local HTTPS and friendly DNS names for services such as Immich, Jellyfin and Audiobookshelf.
- Adding container health checks, bounded logs and monitoring with WUD and Gotify.
- Building safer backup routines with encrypted Restic configuration backups and restore-focused documentation.
- Creating desktop workflows for focus time, meetings, virtual machines and end-of-day recovery.
- Writing reusable skills so hard-won procedures are available in later sessions.
- Inspecting and improving this website, including the backup learning series and this homelab showcase.

## The important habit: verify everything

An agent should not claim a change worked because a command returned successfully. For system changes, I want a read-back or health check. For backups, I want restore tests. For website changes, I want a real Hugo build and a check of the generated pages.

That approach makes Hermes less like a chatbot and more like a careful technical colleague: it can move quickly, but the evidence still comes from the system being changed.

## What comes next

I am continuing to document the lab, add photographs and diagrams, and turn recurring maintenance into small, reviewable workflows. The goal is not to automate everything. It is to make the useful and repetitive parts safer while keeping important decisions visible.

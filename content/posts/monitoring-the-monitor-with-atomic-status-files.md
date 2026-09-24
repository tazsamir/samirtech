---
title: "Monitoring the Monitor with Atomic Status Files"
date: 2026-09-24T00:00:00+01:00
draft: true
description: "How a health script can run successfully while publishing an empty report, and how temporary files, validation and freshness checks prevent it."
tags: [monitoring, linux, reliability, automation, homelab]
---

A monitoring script can be running on schedule and still leave you with no trustworthy monitoring data.

I hit a simple example: the process that generated a JSON health report started correctly, but the published report was zero bytes. Anything reading that file either failed to parse it or had no state to display. The scheduler was healthy. The monitoring output was not.

The cause was a common shell pattern:

```sh
health-report > last-report.json
```

The shell opens `last-report.json` and truncates it *before* `health-report` starts. If the command crashes, is killed or fails while writing, the previous valid report is already gone.

## The final filename is an interface

A status file is not merely a log. Other programs treat its name as an interface: dashboards read it, alerting scripts parse it and people use it during incidents.

That means readers should see one of two things:

1. the complete previous report; or
2. the complete new report.

They should never see a half-written transition between them.

The standard fix is to write beside the final file and rename only after validation:

```sh
set -eu

state_dir="$HOME/.local/state/infra-health"
final="$state_dir/last-report.json"
tmp=$(mktemp "$state_dir/.last-report.XXXXXX")
trap 'rm -f "$tmp"' EXIT

health-report > "$tmp"
test -s "$tmp"
python3 -m json.tool "$tmp" >/dev/null
chmod 600 "$tmp"
mv -f "$tmp" "$final"
trap - EXIT
```

On the same filesystem, `rename` is atomic. Readers see the old inode until the new one replaces it; they do not observe a partially written report.

Creating the temporary file in the destination directory is important. A move across filesystems may fall back to copy-and-delete semantics and lose the atomic guarantee.

## Validate meaning, not only syntax

Non-empty valid JSON is the minimum gate, not always the complete one. A report can be syntactically valid and operationally useless.

For a richer report I also validate required fields:

```python
import json
from pathlib import Path

p = Path("candidate.json")
data = json.loads(p.read_text())

required = {"generated_at", "overall_state", "checks"}
missing = required - data.keys()
if missing:
    raise SystemExit(f"missing fields: {sorted(missing)}")
if not isinstance(data["checks"], list):
    raise SystemExit("checks must be a list")
```

The producer should exit non-zero when it cannot produce a complete report. A wrapper can then preserve the last good file and alert on the failed attempt.

## Keep generation state separate from observed state

The report tells me what the checks found. It should not be the only evidence that report generation itself is working.

I track three independent signals:

- **content:** the latest valid report says healthy or unhealthy;
- **freshness:** its generation time is recent enough for the schedule;
- **execution:** the scheduler last ran the producer and recorded its exit status.

A healthy but stale file is a monitoring failure. So is a fresh file that omits required checks. And a failed producer should not replace yesterday's useful report with an empty one.

A simple freshness probe might look like this:

```python
from pathlib import Path
import time

report = Path("last-report.json")
max_age = 20 * 60
age = time.time() - report.stat().st_mtime
if age > max_age:
    raise SystemExit(f"report is stale: {age:.0f}s")
```

The threshold must match the real schedule and allow a sensible grace period. A five-minute freshness threshold for an hourly job creates noise, while a ten-day threshold for a daily job hides failures.

## Do not destroy the evidence on failure

When generation fails, I keep both kinds of evidence:

- the previous valid report remains at the stable path;
- a small failure record captures when the attempt failed and the command's exit status.

I do not copy raw command output into alerts by default. Health commands can accidentally expose paths, hostnames or credentials. A concise allowlisted summary is safer, with detailed logs retained privately and with bounded size.

The wrapper can also record the failure atomically:

```sh
if ! health-report > "$tmp"; then
    printf '{"failed_at":"%s","exit":1}\n' "$(date --iso-8601=seconds)" \
        > "$state_dir/.last-failure.tmp"
    mv -f "$state_dir/.last-failure.tmp" "$state_dir/last-failure.json"
    exit 1
fi
```

In production I pass through the real exit status and validate this JSON too. The shortened example shows the separation: failed attempt metadata does not overwrite the last known-good health report.

## Test the failure path deliberately

A success-only test misses the problem this design is meant to solve. I test at least four cases in a scratch directory:

1. the producer writes valid JSON and the final file changes;
2. the producer exits non-zero and the prior report remains byte-for-byte identical;
3. the producer writes empty output and validation rejects it;
4. the producer writes malformed JSON and validation rejects it.

I also run a reader repeatedly while replacing the file, checking that it only ever parses the old or new complete document.

Finally, I let the real scheduler run once and confirm the final file's timestamp and content changed. A manual run proves the script; a scheduled run proves the installed path, environment and permissions.

## The rule I now use

Anything consumed under a stable filename is published data. Generate it privately, validate it, then replace it atomically.

That rule applies to health reports, backup manifests, checksums, generated configuration, feed caches and dashboard data. The implementation is small, but it changes a monitoring system from “the command probably ran” into an interface that fails without destroying its last useful answer.
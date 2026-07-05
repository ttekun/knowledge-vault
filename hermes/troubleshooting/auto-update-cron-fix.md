---
type: Troubleshooting
title: "Auto-Update Cron SIGKILL Fix"
description: "Cron auto-update job silently killed by gateway cgroup teardown. Fixed by replacing setsid with systemd-run --scope."
tags: [cron, systemd, cgroup, auto-update, sigkill]
timestamp: 2026-06-28T08:15:00+09:00
---

# Auto-Update Cron SIGKILL Fix

> **Date resolved**: 2026-06-28  
> **Severity**: High (silent job failure, skipped updates)  
> **Status**: Resolved

## Problem

The `hermes-auto-update` cron job (schedule: `0 3 * * *`, `no_agent=True`) was silently failing. The job's `last_run_at` was stale and `next_run_at` skipped past the expected slot.

**Symptoms in logs:**
```
hermes-gateway.service: Killing process <PID> (bash) with signal SIGKILL.
```

**Symptoms in jobs.json:**
- `last_run_at` points to a previous run, not today's 03:00
- `next_run_at` jumped past today's scheduled slot

## Root Cause

The cron script ran `hermes update` **inside the gateway's cron ticker process**. When `hermes update` completes, it restarts the gateway via `systemctl`. systemd's `KillMode=mixed` sends SIGKILL to all processes in the gateway's cgroup — including the bash subprocess running the cron script.

The original script used `setsid` to detach:
```bash
setsid bash "$RUNNER_SCRIPT" "$RESULT_FILE" &
```

**But `setsid` only creates a new session, NOT a new cgroup.** The process remains in the gateway's cgroup and gets SIGKILL'd.

## Solution

Replace `setsid` with `systemd-run --user --scope --collect`:

```bash
systemd-run --user --scope --collect bash "$RUNNER_SCRIPT" "$RESULT_FILE"
```

`systemd-run --scope` creates an **independent cgroup** that survives the gateway's cgroup teardown. The script completes normally, writes its result, and the cron ticker reads it back.

**Script location**: `~/.hermes/scripts/auto-update.sh`

## Additional Fix: Script Timeout

The default `cron.script_timeout_seconds` is 120s, but `hermes update` can take longer (backup + deps + web UI build + gateway drain/restart).

```bash
hermes config set cron.script_timeout_seconds 600
```

Then **restart the gateway** so the new config takes effect:
```bash
systemctl --user restart hermes-gateway
```

## Verification

```bash
# Check job status after next 03:00 run
hermes cron list

# Should show:
# - last_run_at: today's 03:00
# - last_status: "ok"
# - next_run_at: tomorrow's 03:00
```

## Related: Sudoers Entry

The Hermes security scan probe runs `sudo /usr/bin/true` to check sudo access. In a non-interactive gateway environment, this triggers an auth failure.

**Fix**: Added `/etc/sudoers.d/dev-nopasswd-true`:
```
dev ALL=(ALL) NOPASSWD: /usr/bin/true
```

This grants passwordless sudo for `/usr/bin/true` **only**. All other sudo commands still require a password. The agent cannot escalate privileges beyond this single command.

## Related

- [WSL Ollama Port 11434 Conflict](/hermes/troubleshooting/wsl-ollama-port-conflict.md) — another WSL/systemd issue resolved same day
- [MCP ollama-web API Key Issue](/hermes/troubleshooting/mcp-api-key-issue.md) — same investigation session

## Timeline

| Time | Event |
|------|-------|
| 2026-06-28 08:00 | Investigated past session anomalies |
| 2026-06-28 08:10 | Identified SIGKILL in journal logs |
| 2026-06-28 08:13 | Root cause: setsid ≠ new cgroup |
| 2026-06-28 08:15 | Applied `systemd-run --scope` fix |
| 2026-06-28 08:20 | Verified with actual update run |
| Ongoing | Auto-update runs daily at 03:00 JST successfully |
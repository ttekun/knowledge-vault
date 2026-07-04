# WSL Ollama Port 11434 Conflict

> **Date resolved**: 2026-06-28  
> **Severity**: High (service loop, resource waste)  
> **Status**: Resolved (disabled + masked)

## Problem

Linux-side `ollama.service` entered an infinite restart loop, attempting to bind to port 11434 every ~3 seconds. Windows-side Ollama had already claimed port 11434 via WSL2 mirrored networking.

**Impact**: ~8,596 restart attempts in a single day (approximately 14,000 over 30 hours). Constant CPU thrashing and journal log flooding.

## Root Cause

WSL2 mirrored networking mode shares the network namespace between Windows and Linux. Windows Ollama binds `0.0.0.0:11434`, leaving no port for the Linux service. The Linux `ollama.service` (systemd) fails to bind, crashes, and systemd's `Restart=always` policy immediately retries.

## Solution

```bash
# Stop and disable the Linux ollama service
sudo systemctl disable --now ollama

# Mask it to prevent any accidental re-enablement
sudo systemctl mask ollama
```

**Verification:**
```bash
# Confirm service is masked (should show "masked")
systemctl is-enabled ollama 2>/dev/null || echo "masked/inactive"

# Confirm port 11434 is held by Windows Ollama
sudo lsof -i :11434
# or
sudo ss -tlnp sport = :11434
```

## Post-WSL Reboot Check

**Critical**: Re-verify after every WSL reboot. WSL updates or Windows restarts can theoretically reset masked services (rare but documented in some WSL versions).

```bash
# Quick check — should output nothing or "masked"
systemctl is-active ollama 2>/dev/null
```

If the service somehow reactivates:
```bash
sudo systemctl disable --now ollama
sudo systemctl mask ollama
```

## Environment

| Item | Value |
|------|-------|
| OS | WSL2 (Windows 11) |
| Networking mode | Mirrored |
| Windows Ollama | Runs on port 11434 (desired) |
| Linux Ollama | Disabled + masked (desired) |
| Date disabled | 2026-06-28 |

## Notes

- Windows-side Ollama is the canonical instance — it serves the MCP `ollama-web` integration via `OLLAMA_API_KEY`.
- The Linux `ollama` package can remain installed; only the systemd service needs to be disabled.
- If WSL networking mode changes from mirrored to NAT, this conflict may not occur (Linux would get its own IP). But mirrored mode is preferred for other reasons.
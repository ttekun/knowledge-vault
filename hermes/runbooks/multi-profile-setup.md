---
type: Runbook
title: "Multi-Profile Setup"
description: "Two independent Hermes Agent profiles on one WSL2 host with separate Telegram bots and systemd gateway services."
tags: [hermes, multi-profile, telegram, systemd, gateway]
timestamp: 2026-06-21T14:56:00+09:00
---

# Multi-Profile Setup Runbook

> **Date created**: 2026-06-21  
> **Last verified**: 2026-06-28  
> **Status**: Operational

## Overview

Two independent Hermes Agent profiles running on the same WSL2 host, each with its own Telegram bot and systemd gateway service. Both share the same model configuration, SOUL.md, MEMORY.md, and USER.md.

## Profile Configuration

| Property | Default | Second |
|----------|---------|--------|
| Profile name | `default` | `second` |
| Telegram bot | (primary bot) | `s_202606211456_bot` |
| TELEGRAM_ALLOWED_USERS | `1316005593` | `7429185441` |
| systemd service | `hermes-gateway.service` | `hermes-gateway-second.service` |
| CLI wrapper | `hermes` | `~/.local/bin/second` or `hermes -p second` |
| Model | `ollama-launch/glm-5.2:cloud` | `ollama-launch/glm-5.2:cloud` |
| SOUL.md | Shared | Shared (copied) |
| MEMORY.md | Shared | Shared (copied) |
| USER.md | Shared | Shared (copied) |

## Setup Steps

### 1. Create the second profile

```bash
hermes profile create second --clone-all
```

`--clone-all` copies config, skills, cron, memories, and plugins from the default profile.

### 2. Configure the second Telegram bot

```bash
hermes -p second gateway setup
```

- Use a separate Telegram bot token (create via @BotFather)
- Set `TELEGRAM_ALLOWED_USERS` to the second user's Telegram ID
- Set `TELEGRAM_BOT_TOKEN` in the profile's `.env`

### 3. Install the second gateway as a systemd service

```bash
hermes -p second gateway install
```

This creates `hermes-gateway-second.service` under `~/.config/systemd/user/`.

### 4. Enable linger (survives SSH/logout)

```bash
sudo loginctl enable-linger $USER
```

This is already set — required for both gateway services to persist.

### 5. Create CLI wrapper (optional convenience)

```bash
cat > ~/.local/bin/second << 'EOF'
#!/bin/bash
exec hermes -p second "$@"
EOF
chmod +x ~/.local/bin/second
```

### 6. Start and verify

```bash
systemctl --user start hermes-gateway-second
systemctl --user status hermes-gateway-second
```

## Verification

```bash
# Both gateways running
systemctl --user status hermes-gateway hermes-gateway-second

# Both respond to messages
hermes -p second chat -q "ping"
hermes chat -q "ping"
```

## Related

- [WSL Ollama Port 11434 Conflict](/hermes/troubleshooting/wsl-ollama-port-conflict.md) — both gateways affected by port conflict
- [MCP ollama-web API Key Issue](/hermes/troubleshooting/mcp-api-key-issue.md) — MCP server runs under both profiles

## Maintenance Notes

- **SOUL.md / MEMORY.md / USER.md**: Shared between profiles. When updated in one, copy to the other:
  ```bash
  cp ~/.hermes/SOUL.md ~/.hermes/profiles/second/SOUL.md
  cp ~/.hermes/MEMORY.md ~/.hermes/profiles/second/MEMORY.md
  cp ~/.hermes/USER.md ~/.hermes/profiles/second/USER.md
  ```
  Then restart the second gateway: `systemctl --user restart hermes-gateway-second`

- **Skills**: Each profile has its own `skills/` directory. Skills created in one profile do not automatically appear in the other. Copy manually if needed.

- **Cron jobs**: Each profile has its own cron job store. The auto-update job only needs to run in one profile (default).

- **Independent sessions**: Each profile has its own session database. Session search in one profile does not see the other's sessions.

## Troubleshooting

| Problem | Solution |
|---------|----------|
| Second bot not responding | `systemctl --user status hermes-gateway-second` — check for errors |
| Both bots respond to same user | Check `TELEGRAM_ALLOWED_USERS` — each must have its own allowed list |
| Config changes not reflected | `systemctl --user restart hermes-gateway-second` |
| Profile command not found | Ensure `~/.local/bin` is in `$PATH` |
# Knowledge Vault

Personal knowledge repository — deep research reports, notes, and runbooks.

## Index

### AI / Agents
| Date | Title | Confidence |
|------|-------|------------|
| 2026-07-04 | [Loop Engineering — Deep Research](ai/loop-engineering-research.md) | 🟡 |

### Hermes Agent Operations

#### Troubleshooting
| Date | Title | Description |
|------|-------|-------------|
| 2026-06-28 | [WSL Ollama Port 11434 Conflict](hermes/troubleshooting/wsl-ollama-port-conflict.md) | Linux ollama.service restart loop (8,596×/day) due to WSL mirrored networking port conflict |
| 2026-06-28 | [Auto-Update Cron SIGKILL Fix](hermes/troubleshooting/auto-update-cron-fix.md) | setsid → systemd-run --scope to survive gateway cgroup teardown |
| 2026-06-28 | [MCP ollama-web API Key Issue](hermes/troubleshooting/mcp-api-key-issue.md) | _build_safe_env() strips API keys from MCP subprocess; fix via config env block |

#### Runbooks
| Date | Title | Description |
|------|-------|-------------|
| 2026-06-21 | [Multi-Profile Setup](hermes/runbooks/multi-profile-setup.md) | Two Telegram bots on one WSL host with isolated profiles + shared SOUL/MEMORY |

---

## Conventions

- **File naming**: `kebab-case.md` under domain folder
- **Confidence labels**: ✅ Verified / 🟡 Partially verified / ⚠️ Unverified / 🔴 Rumor
- **Source tiers**: Tier 1 (official) > Tier 2 (wire) > Tier 3 (major media) > Tier 4 (analyst) > Tier 5 (blog/SNS)
- **Each report must include**: Source summary table + verification limitations disclosure
- **Index updated automatically** when new reports are added

## Structure

```
knowledge-vault/
├── README.md            ← this file (auto-updated index)
├── ai/                  ← AI, LLM, agents, prompt engineering
└── hermes/              ← Hermes Agent operations, troubleshooting, runbooks
```
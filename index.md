---
okf_version: "0.1"
---

# Knowledge Vault

Personal knowledge repository — deep research reports, notes, and runbooks.
OKF v0.1 conformant bundle.

## AI / Agents

* [Loop Engineering — Deep Research](ai/loop-engineering-research.md) - In-depth survey of Loop Engineering paradigm: 5 components, verifier bottleneck, Andrew Ng's 3-loop model.
* [AI / Agent Trends — Weekly Radar 2026-07-06](ai/2026-07-06-ai-agent-trends.md) - Fable 5 export-control lift, GPT-5.6 preview, loop engineering consolidation.
* [AI / Agent Trends — Weekly Radar 2026-07-13](ai/2026-07-13-ai-agent-trends.md) - GPT-5.6 GA launch, Meta Muse Spark 1.1, GPT-Live voice, Gemini Managed Agents upgrade, harness engineering consolidation.

## Hermes Agent Operations

### Troubleshooting

* [WSL Ollama Port 11434 Conflict](hermes/troubleshooting/wsl-ollama-port-conflict.md) - Linux ollama.service restart loop due to WSL2 mirrored networking port conflict.
* [Auto-Update Cron SIGKILL Fix](hermes/troubleshooting/auto-update-cron-fix.md) - setsid → systemd-run --scope to survive gateway cgroup teardown.
* [MCP ollama-web API Key Issue](hermes/troubleshooting/mcp-api-key-issue.md) - _build_safe_env() strips API keys from MCP subprocess; fix via config env block.
* [ctx Semantic Search on WSL glibc 2.35](hermes/troubleshooting/ctx-semantic-search-glibc-fix.md) - Build ctx with ort-load-dynamic + runtime ONNX Runtime library to enable semantic search without OS upgrade.

### Runbooks

* [Multi-Profile Setup](hermes/runbooks/multi-profile-setup.md) - Two Telegram bots on one WSL host with isolated profiles + shared SOUL/MEMORY.

## Conventions

- **OKF conformant**: Every concept .md has YAML frontmatter with required `type` field.
- **Confidence labels**: ✅ Verified / 🟡 Partially verified / ⚠️ Unverified / 🔴 Rumor.
- **Source tiers**: Tier 1 (official) > Tier 2 (wire) > Tier 3 (major media) > Tier 4 (analyst) > Tier 5 (blog/SNS).
- **Privacy policy**: Non-private content only. No personal identifying data.
- **See also**: [Change log](log.md)
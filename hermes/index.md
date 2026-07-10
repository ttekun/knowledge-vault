# Hermes Agent Operations

## Troubleshooting

* [WSL Ollama Port 11434 Conflict](troubleshooting/wsl-ollama-port-conflict.md) - Linux ollama.service restart loop due to WSL2 mirrored networking port conflict.
* [Auto-Update Cron SIGKILL Fix](troubleshooting/auto-update-cron-fix.md) - setsid → systemd-run --scope to survive gateway cgroup teardown.
* [MCP ollama-web API Key Issue](troubleshooting/mcp-api-key-issue.md) - _build_safe_env() strips API keys from MCP subprocess; fix via config env block.
* [ctx Semantic Search on WSL glibc 2.35](troubleshooting/ctx-semantic-search-glibc-fix.md) - Build ctx with ort-load-dynamic + runtime ONNX Runtime library to enable semantic search without OS upgrade.

## Runbooks

* [Multi-Profile Setup](runbooks/multi-profile-setup.md) - Two Telegram bots on one WSL host with isolated profiles + shared SOUL/MEMORY.
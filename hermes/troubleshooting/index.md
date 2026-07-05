# Troubleshooting

* [WSL Ollama Port 11434 Conflict](wsl-ollama-port-conflict.md) - Linux ollama.service restart loop due to WSL2 mirrored networking port conflict.
* [Auto-Update Cron SIGKILL Fix](auto-update-cron-fix.md) - setsid → systemd-run --scope to survive gateway cgroup teardown.
* [MCP ollama-web API Key Issue](mcp-api-key-issue.md) - _build_safe_env() strips API keys from MCP subprocess; fix via config env block.
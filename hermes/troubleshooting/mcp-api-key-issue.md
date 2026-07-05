---
type: Troubleshooting
title: "MCP ollama-web API Key Issue"
description: "Hermes _build_safe_env() strips API keys from MCP subprocess env. Fix via explicit env block in config.yaml."
tags: [mcp, ollama-web, api-key, config, env]
timestamp: 2026-06-28T08:13:00+09:00
---

# MCP ollama-web API Key Issue

> **Date resolved**: 2026-06-28  
> **Severity**: Medium (MCP web_search/web_fetch non-functional)  
> **Status**: Resolved

## Problem

The MCP server `ollama-web` (providing `web_search` and `web_fetch` tools) failed with authentication errors. The `OLLAMA_API_KEY` environment variable was not reaching the MCP subprocess.

## Root Cause

Hermes Agent's `tools/mcp_tool.py` contains a `_build_safe_env()` function that constructs the environment for MCP subprocesses. For security, it only passes a whitelist of "safe" variables (`PATH`, `HOME`, `LANG`, etc.) and **excludes API keys and other secrets**.

Even though `OLLAMA_API_KEY` was defined in `~/.hermes/.env`, it was never passed to the MCP subprocess because `_build_safe_env()` filtered it out.

## Solution

Add an explicit `env` block in `config.yaml` under the MCP server definition:

```yaml
mcp_servers:
  ollama-web:
    command: ~/.hermes/ollama-mcp-venv/bin/python
    args:
      - ~/.hermes/scripts/ollama-web-mcp.py
    env:
      OLLAMA_API_KEY: ${OLLAMA_API_KEY}
```

The `${OLLAMA_API_KEY}` interpolation reads from `~/.hermes/.env` and explicitly injects the key into the MCP subprocess environment, bypassing the `_build_safe_env()` filter.

## Verification

```bash
# Test MCP connection
hermes mcp test ollama-web

# In a session, use the tools
# web_search via mcp_ollama_web_web_search
# web_fetch via mcp_ollama_web_web_fetch
```

## Related

- [Auto-Update Cron SIGKILL Fix](/hermes/troubleshooting/auto-update-cron-fix.md) — same investigation session, both fixed 2026-06-28
- [Multi-Profile Setup](/hermes/runbooks/multi-profile-setup.md) — MCP server runs under both profiles

## Architecture

| Component | Path |
|-----------|------|
| MCP script | `~/.hermes/scripts/ollama-web-mcp.py` |
| Python venv | `~/.hermes/ollama-mcp-venv/` |
| Config | `~/.hermes/config.yaml` → `mcp_servers.ollama-web` |
| API key | `~/.hermes/.env` → `OLLAMA_API_KEY` |

## General Pattern

This fix applies to **any MCP server that needs API keys**. The pattern is:

1. Define the key in `~/.hermes/.env`
2. Add `env: { KEY_NAME: ${KEY_NAME} }` to the server's config block
3. Do NOT rely on the key being inherited from the parent process — `_build_safe_env()` will strip it

## Notes

- The `ollama-web` MCP server provides Cloudflare-bypassing web search and page content extraction using a free Ollama account.
- It serves as an alternative to Tavily/Firecrawl/Exa for `web_extract` when `web.extract_backend` is unset.
- Tools exposed: `mcp_ollama_web_web_search`, `mcp_ollama_web_web_fetch`
# Kola marketplace

A Claude Code [plugin marketplace](https://docs.anthropic.com/en/docs/claude-code/plugins)
for [Kola](https://getkola.app) — the AI-native people-memory layer.

## Plugins

| Plugin | What it adds |
|---|---|
| **[kola](./kola)** | Use Kola from Claude Code — meeting prep, network search, contact capture, list management, custom-field schema, a cross-channel recent-activity view, and a `contact-suggester` agent that proactively surfaces people from your network when you discuss real work in chat. Talks to the Kola desktop app's local MCP server. |

## Install

```bash
# In Claude Code
/plugin marketplace add <path-to-this-repo>
/plugin install kola@kola-marketplace
```

Kola.app must be running for the plugin to work — its MCP server is
local-only (`127.0.0.1:47900`). See the plugin's
[README](./kola/README.md) for details.

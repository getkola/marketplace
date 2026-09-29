# Kola marketplace

A Claude Code [plugin marketplace](https://docs.anthropic.com/en/docs/claude-code/plugins)
for [Kola](https://getkola.app) — the AI-native people-memory layer.

## Plugins

| Plugin | What it adds |
|---|---|
| **[kola](./kola)** | Use Kola from Claude Code — meeting prep, network search, contact capture, list management, custom-field schema, a cross-channel recent-activity view, and a `contact-suggester` agent that proactively surfaces people from your network when you discuss real work in chat. Talks to Kola through its remote MCP endpoint, answered by the Kola app on your Mac. |

## Install

```bash
# In Claude Code
/plugin marketplace add <path-to-this-repo>
/plugin install kola@kola-marketplace
```

The plugin connects to `https://mcp.getkola.app/mcp`, which answers from
the Kola app on your Mac. You need a Kola account, remote access switched
on, and Kola running on the Mac. A local-only setup without remote access
is also possible. See the plugin's [README](./kola/README.md) for both.

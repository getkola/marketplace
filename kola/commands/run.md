---
description: Start Kola.app on macOS if it isn't running, and explain what to fix when Kola's remote MCP server refuses a call.
---

# /kola:run

The plugin talks to Kola through the remote endpoint
`https://mcp.getkola.app/mcp`. That endpoint answers from the Kola app on
the user's Mac, through a tunnel the app opens. So Kola.app must be running,
and remote access must be switched on. This command starts the app and
explains the setup. macOS only.

## Instructions

1. **Platform guard.** Run `uname -s`. If the output is not `Darwin`, stop and tell the user this command only supports macOS — they should start Kola manually on other platforms.

2. **Check whether Kola is running.** The app still serves its local MCP server on `127.0.0.1:47900`, so a local probe is the quickest check:

   ```bash
   curl -sS -o /dev/null -w '%{http_code}' --max-time 2 http://127.0.0.1:47900/mcp
   ```

   Any HTTP status code (including 4xx/5xx) means the app is up. Skip to step 5.

3. **Launch Kola.app.** Run:

   ```bash
   open -a Kola
   ```

   If `open` exits non-zero (typically "Unable to find application named 'Kola'"), tell the user Kola.app isn't installed and point them at <https://getkola.app>. Do not retry.

4. **Wait for the app.** Poll the local endpoint up to 30 times, one second apart:

   ```bash
   for i in $(seq 1 30); do code=$(curl -sS -o /dev/null -w '%{http_code}' --max-time 1 http://127.0.0.1:47900/mcp || echo ""); if [ -n "$code" ] && [ "$code" != "000" ]; then echo "ready after ${i}s (HTTP $code)"; exit 0; fi; sleep 1; done; exit 1
   ```

   On timeout, report that Kola was launched but did not come up within 30s.

5. **Explain the remote setup.** A running app is not enough. The remote endpoint checks these things in order, and each has its own error text. If a Kola tool call failed, match its error message to this table and give the user the fix:

   | Error text from the tool call | Fix |
   |---|---|
   | `missing authorization header` / `invalid access token` / `This client is no longer connected` | Run `/mcp` in Claude Code, pick the `kola` server, and choose Authenticate. Sign in with the Kola account and approve. |
   | `Remote access is off for this Kola account` | Open <https://my.getkola.app/settings> and turn remote access on. |
   | `no client is allowed` / `This address is not on the allow list` | At <https://my.getkola.app/settings>, turn on **Any address**. Claude Code calls from the user's own internet address, not from Anthropic's servers, so the "Claude" option does not cover it. |
   | `Kola is not connected` / `disconnected while answering` / `did not answer in time` | In Kola on the Mac, open Settings → MCP and turn on **Allow remote access**. Keep the Mac awake. If two Macs have it on, the one that connected last answers. |

   If no tool call has failed yet, list the three switches briefly (sign-in, account remote access with Any address, the Mac's Allow remote access) and stop.

6. **Don't restart MCP clients.** This command only starts the app and explains. The tools are named `mcp__plugin_kola_kola__*` when installed through this plugin, and `mcp__kola__*` only when the server comes from a project `.mcp.json` entry called `kola`; the prefix is chosen by the client, not by Kola.

---
description: Start Kola.app on macOS if its local MCP server isn't already responding on 127.0.0.1:47900.
---

# /kola:run

Make sure Kola.app is running so the local MCP server at `127.0.0.1:47900` is reachable. macOS only.

## Instructions

1. **Platform guard.** Run `uname -s`. If the output is not `Darwin`, stop and tell the user this command only supports macOS — they should start Kola manually on other platforms.

2. **Probe the MCP endpoint first.** Run:

   ```bash
   curl -sS -o /dev/null -w '%{http_code}' --max-time 2 http://127.0.0.1:47900/mcp
   ```

   Any HTTP status code (including 4xx/5xx) means the server is up. If the probe succeeds, report `Kola is already running.` and stop — do not launch a second instance.

3. **Launch Kola.app.** Run:

   ```bash
   open -a Kola
   ```

   If `open` exits non-zero (typically "Unable to find application named 'Kola'"), tell the user Kola.app isn't installed and point them at <https://getkola.app>. Do not retry.

4. **Wait for the MCP server.** Poll the endpoint up to 30 times, one second apart:

   ```bash
   for i in $(seq 1 30); do
     code=$(curl -sS -o /dev/null -w '%{http_code}' --max-time 1 http://127.0.0.1:47900/mcp || echo "")
     if [ -n "$code" ] && [ "$code" != "000" ]; then
       echo "ready after ${i}s (HTTP $code)"
       exit 0
     fi
     sleep 1
   done
   exit 1
   ```

   On success, report `Kola started — MCP server is up.` On timeout, report that Kola was launched but the MCP server didn't come up within 30s, and suggest the user check Kola's Settings → MCP.

5. **Don't restart MCP clients.** This command only starts the app. If Claude Code's MCP client connected before Kola was up, the user may need to reload the session for `mcp__kola__*` tools to appear — mention this only if step 4 had to launch the app (not when it was already running).

# Connecting an agent to Sketch

1. Use Sketch 2025.2.4 or later, downloaded from sketch.com. The Mac App Store build has no MCP server.
2. In Sketch, press ⌘K, type "MCP", and choose **Start MCP Server**. Alternatively, go to Settings → General → MCP Server. If macOS asks, allow Local Network access.
3. Point the agent at `http://localhost:31126/mcp`. The bottom of the Sketch window shows the actual port.

| Agent | How to connect |
|---|---|
| Claude Code (also when run inside T3 Code) | `claude mcp add --scope user --transport http sketch http://localhost:31126/mcp`, then check with `claude mcp get sketch` |
| Codex (also in T3 Code) | Add to `~/.codex/config.toml`:<br>`[mcp_servers.sketch]`<br>`url = "http://localhost:31126/mcp"` |
| Cursor | Add to `mcp.json`: `{ "mcpServers": { "sketch": { "type": "http", "url": "http://localhost:31126/mcp" } } }` |
| Claude desktop | Settings → Extensions → Browse → "Sketch" → Install |

After connecting, start a new agent session so the tools load.

Check that it works: call `get_guide` (topic `mcp`), then `get_document_info`. Both should succeed while a document is open.

## Troubleshooting

- Connection refused: the server isn't running. It does not restart automatically when Sketch relaunches.
- The server won't start: in System Settings → Privacy & Security → Local Network, turn Sketch off and back on, then relaunch Sketch.
- The port is taken: quit Sketch, run `defaults write com.bohemiancoding.sketch3 mcpServerPortNumber -int <1024–49151>`, relaunch Sketch, and update the URL in your agent's config.
- To point the agent at a specific layer: select it in Sketch, choose **Copy Layer ID** from the Command Bar or the right-click Copy menu, and paste the ID into your prompt.

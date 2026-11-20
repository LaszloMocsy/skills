# Connecting an agent to Sketch

1. Use Sketch 2026.4 or later for the full skill (Components with Variants, Sections, comment tools), downloaded from sketch.com. 2025.2.4 is the minimum for the MCP server at all. The Mac App Store build has no MCP server. Sketch 2026.4 requires macOS 15 Sequoia or later, and documents saved with it don't open in older versions.
2. In Sketch, press ⌘K, type "MCP", and choose **Start MCP Server**. Alternatively, go to Settings → General → MCP Server. If macOS asks, allow Local Network access.
3. Point the agent at the URL Sketch shows (default `http://localhost:31126/mcp`).

| Agent | How to connect |
|---|---|
| Claude Code (also when run inside T3 Code) | `claude mcp add --scope user --transport http sketch http://localhost:31126/mcp`, then check with `claude mcp get sketch` |
| Codex (also in T3 Code) | Add to `~/.codex/config.toml`:<br>`[mcp_servers.sketch]`<br>`url = "http://localhost:31126/mcp"` |
| Cursor | Add to `mcp.json`: `{ "mcpServers": { "sketch": { "type": "http", "url": "http://localhost:31126/mcp" } } }` |
| Claude desktop | Settings → Extensions → Browse → "Sketch" → Install |

After connecting, start a new agent session so the tools load.

Check that it works: call `get_guide` (topic `mcp`), then `get_document_info`. Both should succeed while a document is open.

## Tools (Sketch 2026.4)

| Tool | Purpose |
|---|---|
| `get_guide` | Version-matched guides: `mcp`, `use`, `layout`, `organization`, `styling`, `symbols`, `assets`, `prototyping`, `troubleshooting` |
| `get_document_info` | Pages, top-level frames (with their section), sections |
| `get_layer_tree_summary` | Compact hierarchy with Stack, Section, Variant and Instance details |
| `get_libraries`, `get_design_assets` | Libraries and their components (incl. Components with Variants), styles, Color Variables, templates |
| `get_symbol_overrides` | Overrides of a component instance |
| `get_screenshot` | PNG of a layer, frame or page (max 100 megapixels) |
| `run_code` | Runs SketchAPI JavaScript |
| `list_comments`, `save_comment`, `delete_comment` | Workspace comment threads |
| `list_notifications` | Workspace notifications (comments, mentions, invites) |

Comment and notification tools need Sketch signed in to a Sketch account and a document saved to a Workspace.

## Troubleshooting

- Connection refused: the server isn't running. It does not restart automatically when Sketch relaunches.
- Tools fail after Sketch was restarted: the server's session changed. Restart the agent / MCP client; if that doesn't help, fully quit and relaunch Sketch, start the server again, and retry once.
- The server won't start: in System Settings → Privacy & Security → Local Network, turn Sketch off and back on, then relaunch Sketch.
- The port is taken: quit Sketch, run `defaults write com.bohemiancoding.sketch3 mcpServerPortNumber -int <1024–49151>`, relaunch Sketch, and update the URL in your agent's config.
- "Comments are only available in documents saved to a Workspace" / "Not signed in to a Sketch account": save the document to a Workspace / sign in under Sketch → Settings → Account.
- To point the agent at a specific layer: select it in Sketch, choose **Copy Layer ID** from the Command Bar or the right-click Copy menu, and paste the ID into your prompt. Sketch share links (`https://sketch.com/s/<doc>/f/<frame>`) work too.

## Re-testing after a Sketch update

The gotchas and recipes are verified against one Sketch build (currently 2026.4, 238217). After a Sketch update, rerun these checks before trusting them. Work in a scratch document (`new sketch.Document()`, resolve it by ID, close it without saving at the end), never in the user's file.

1. Version probe, then compare `get_guide` topics and the MCP tool list with the tables above. Read the new release's API notes: https://developer.sketch.com/plugins/updates/llms.txt
2. Run each check below. If a result differs, update the matching gotcha in `SKILL.md` and the recipe it affects. Remove a workaround once the bug is fixed.
3. Run recipes 1–14 in order on the scratch document, with a canvas check after each one that changes something visible, plus recipes 16–17 on the result.
4. Update the build number here and the "tested on" line at the top of `recipes.md`.

| Check | How | Result on 2026.4 |
|---|---|---|
| Missing font | Set `fontFamily = 'NoSuchFont'` on a new Text, read it back | Silently keeps Helvetica |
| Font size readback | Read `style.fontSize` on a styled text layer | `null` |
| Text style from a plain object | Push `{ fontSize, fontWeight, textSwatch }` as a shared text style, apply it, read `textSwatch` | Size and swatch kept |
| Frame with `style.fills` | New canvas Frame with one swatch fill; count `style.fills` | 1 (no extra white fill) |
| Frame without fills | New canvas Frame, no style | One white `#ffffffff` fill |
| Components page name | `Page.createSymbolsPage()` in a new document | "Components" |
| Variant property names | `new Group.VariantSet` with `Button/Primary/M`-style sources and no `variantProperties` | "Property 1", "Property 2" |
| Default variant | Read `set.defaultVariantMaster` | First value of each property |
| Section stack collapse | Wrap a top-level frame in a Section, give the section a `stackLayout` | Frame collapses to 0.1 × 0.1 (its sizing is `Relative`) |
| Placement measurement | Frames in a section: compare `page.layers` vs `page.canvasLevelFrames` right edges | `canvasLevelFrames` is short by the section padding |
| Long text in a column Stack | 345-wide column, long Body text, `alignItems: Stretch` | Overflows unless the text is `Fill` |
| Stretched instance | Button instance in a `Stretch` column | Label stays left unless the instance is `Fill` |
| Card row height | Row created with `alignItems: Stretch`, cards with wrapping text | Row stays at its initial height |
| Stack packing order | Row with `layers: [A, B, C]` | Draws C, B, A |
| Section as Start Point | `section.flowStartPoint = true` | Throws "Can't make a Section a prototype start point" |
| Screenshot size limit | `get_screenshot` on a very wide page | Refused above 100 megapixels |

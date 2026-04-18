# room-genie-skills

Claude plugin that connects [Room Genie](https://roomgenie.travel) to Claude Desktop, Claude Code, and other MCP-capable clients.

Bundles:

- The `room-genie` MCP server (hosted at `https://app.roomgenie.travel/api/mcp`) with 10 tools for creating/listing alerts and checking live Disney rates
- 5 skills that teach Claude how to use those tools for:
  - **room-genie-core** — alert CRUD workflows (all products)
  - **disney-world-planner** — Walt Disney World resort + package quotes
  - **disneyland-planner** — Disneyland Resort (California)
  - **aulani-planner** — Aulani (Hawaii)
  - **disney-cruise-planner** — Disney Cruise Line sailings + staterooms

## Install

### Claude Code

```bash
# Add the marketplace
/plugin marketplace add tombandini-7/room-genie-skills

# Install the plugin
/plugin install room-genie@room-genie
```

### Claude Desktop

1. Edit `~/Library/Application Support/Claude/claude_desktop_config.json`
2. Add under `mcpServers`:
   ```json
   {
     "mcpServers": {
       "room-genie": {
         "type": "streamable-http",
         "url": "https://app.roomgenie.travel/api/mcp"
       }
     }
   }
   ```
3. Restart Claude Desktop.
4. The skills in this repo are Claude Code–specific — Claude Desktop gets the MCP tools but not the skill prompts. Roughly the same capability; Claude will reason from the tool descriptions.

## First use

The MCP server is OAuth-protected. On the first tool call, Claude opens a browser window where you sign in with your Room Genie account and approve access. After approval, the token is cached locally by Claude.

To revoke access later, visit [Connected Apps](https://app.roomgenie.travel/settings/connected-apps).

## What you can ask Claude to do

- "List my Room Genie alerts"
- "What's available at the Polynesian Dec 1–5 for 2 adults?"
- "Quote a 5-night WDW package at Grand Floridian with park hoppers for 2 adults and 1 child aged 8"
- "Alert me if the Disney Wish 4-nighter in Sep 2026 drops below $3200"
- "Pause my alert for Aulani"

## Local development

If you're working against a local Room Genie dev server (not the hosted one), edit `.claude-plugin/plugin.json` and change the `url` field to `http://localhost:3000/api/mcp`.

## Requirements

- A Room Genie account (sign up at https://roomgenie.travel)
- An Explorer subscription ($29/mo) for live rate exploration with prices. `availability`-mode checks work on any plan.
- Alert creation consumes credits or requires an active Watcher/Explorer subscription.

## License

Proprietary — © Room Genie.

# room-genie-skills

Claude plugin that teaches Claude how to use the [Room Genie](https://roomgenie.travel) MCP server for Disney hotel and cruise pricing.

Bundles 5 skills:

- **room-genie-core** — alert CRUD workflows (all products)
- **disney-world-planner** — Walt Disney World resort + package quotes
- **disneyland-planner** — Disneyland Resort (California)
- **aulani-planner** — Aulani (Hawaii)
- **disney-cruise-planner** — Disney Cruise Line sailings + staterooms

The MCP server itself lives at `https://app.roomgenie.travel/api/mcp` and is registered separately (see below) because Claude Code's plugin format only supports stdio MCP servers today — not remote HTTP.

## Install (Claude Code)

Two steps — register the server, then install the skills:

```bash
# 1. Register the hosted MCP server (one-time). Opens a browser for Supabase OAuth on first tool call.
claude mcp add --transport streamable-http room-genie https://app.roomgenie.travel/api/mcp

# 2. Add the marketplace + install the skills plugin
/plugin marketplace add tombandini-7/room-genie-skills
/plugin install room-genie@room-genie
```

Verify:
```bash
claude mcp list            # should show room-genie as connected
/plugins                    # should show room-genie enabled with 5 skills
```

## Install (Claude Desktop)

Edit `~/Library/Application Support/Claude/claude_desktop_config.json`:
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

Restart Claude Desktop. The skills in this repo are Claude Code-specific — Claude Desktop gets the MCP tools directly and reasons from the tool descriptions, which covers most workflows.

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

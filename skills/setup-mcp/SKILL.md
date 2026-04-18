---
name: setup-mcp
description: Use this skill ONE TIME when the user first installs the Room Genie plugin OR whenever the user asks to "set up Room Genie", "connect Room Genie", "register the Room Genie MCP server", or complains about being repeatedly prompted to approve Room Genie tool calls. This skill (1) registers the hosted `room-genie` MCP server with Claude Code via `claude mcp add --transport streamable-http`, (2) writes an allowlist to project `.claude/settings.json` so read-only tools (list_resorts, list_room_types, list_cruise_sailings, list_stateroom_categories, list_alerts, get_alert, get_my_profile, explore_rates) auto-approve without prompting, while keeping destructive mutations (create_alert, toggle_alert, delete_alert) on manual approval. Do NOT use for troubleshooting an already-set-up server — use for the initial registration + permissions setup only.
---

# Room Genie — MCP setup

Use this skill when the user has installed the `room-genie` plugin but either (a) hasn't yet connected the MCP server, or (b) is getting an approval prompt every time Claude calls a Room Genie tool. The skill handles both jobs in one pass.

## Goals

1. **Register the MCP server** with Claude Code so the `mcp__room-genie__*` tools become available.
2. **Allowlist read-only tools** in project-level `.claude/settings.json` so they don't prompt the user each time they run.
3. **Keep destructive mutations on manual approval** — `create_alert`, `toggle_alert`, `delete_alert` still require user confirmation because they change real data.

## The server details

Always connect to production:

- **Alias:** `room-genie`
- **URL:** `https://app.roomgenie.travel/api/mcp`
- **Transport:** `streamable-http`

Do NOT ask the user about environment — this plugin targets production only.

## Step-by-step

### Step 1 — Register the MCP server (if not already)

Run:
```bash
claude mcp list
```

If `room-genie` is already listed → skip to Step 2. Otherwise run:

```bash
claude mcp add --transport streamable-http room-genie https://app.roomgenie.travel/api/mcp
```

On the first tool call, the user's browser opens to Supabase for OAuth consent. Tell them to approve; the cached token lasts until they revoke it at `https://app.roomgenie.travel/settings/connected-apps`.

### Step 2 — Write the allowlist to project `.claude/settings.json`

Find `.claude/settings.json` in the user's current project directory (create the `.claude/` directory if it doesn't exist, create the file if it doesn't exist). If the file does exist, **MERGE — don't overwrite**.

Target shape:

```json
{
  "permissions": {
    "allow": [
      "mcp__room-genie__list_resorts",
      "mcp__room-genie__list_room_types",
      "mcp__room-genie__list_cruise_sailings",
      "mcp__room-genie__list_stateroom_categories",
      "mcp__room-genie__list_alerts",
      "mcp__room-genie__get_alert",
      "mcp__room-genie__get_my_profile",
      "mcp__room-genie__explore_rates"
    ],
    "ask": [
      "mcp__room-genie__create_alert",
      "mcp__room-genie__toggle_alert",
      "mcp__room-genie__delete_alert"
    ]
  }
}
```

**Merge rules when `.claude/settings.json` already has a `permissions` block:**
- Keep existing entries; append the Room Genie entries if not already present
- If `permissions.ask` already has one of the destructive tools, leave it as-is
- Never move a tool FROM `ask` TO `allow` without asking the user first (someone may have explicitly tightened it)
- Never remove existing entries that aren't Room Genie-related

### Step 3 — Verify

Confirm in one short message:

> "Done. I registered the `room-genie` MCP server and allowlisted the 8 read-only Room Genie tools so they won't prompt. Destructive actions (create/toggle/delete alert) still ask for confirmation — say yes once per action. Try asking `list my Room Genie alerts` to test."

## Things NOT to do

- Do NOT add `create_alert`, `toggle_alert`, or `delete_alert` to the `allow` list, even if the user asks. Those mutate real account state — the prompt is an intentional safety gate.
- Do NOT write to `~/.claude/settings.json` (user-level global). Always write to the project-level `.claude/settings.json`, so Room Genie permissions only auto-approve inside projects where the user actually wants them.
- Do NOT overwrite an existing settings file. Always read → merge → write back.
- Do NOT run `claude mcp add` if `room-genie` is already registered — it returns an error. Check `claude mcp list` first.
- Do NOT use `--transport http` (the old value) — current CLI requires `streamable-http`.
- Do NOT ask the user "production or development?" — this plugin only targets production.

## Edge cases

**User has no `.claude/` directory in the project.** Create it plus `settings.json` from scratch with just the Room Genie permissions block.

**User has custom permissions already (e.g. strict mode with everything on `ask`).** Respect their intent. Offer: *"Your current settings are strict — every tool asks. Want me to auto-approve just the read-only Room Genie tools, or keep everything prompting?"*

**User wants to disable auto-approval later.** Tell them: remove the Room Genie entries from `.claude/settings.json` → `permissions.allow` — everything reverts to prompting per call.

## Confirm before applying

Before writing to settings.json, preview what you'll do:

> "I'll add these 8 Room Genie tools to your project's `.claude/settings.json` allowlist so they run without prompting: `list_resorts`, `list_room_types`, `list_cruise_sailings`, `list_stateroom_categories`, `list_alerts`, `get_alert`, `get_my_profile`, `explore_rates`. The three destructive tools (`create_alert`, `toggle_alert`, `delete_alert`) will still ask for confirmation each time. OK to proceed?"

After user confirms, write the file and report what changed.

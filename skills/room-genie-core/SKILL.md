---
name: room-genie-core
description: Core Room Genie workflows for creating, listing, toggling, and deleting alerts across Walt Disney World, Disneyland Resort, Aulani, and Disney Cruise Line. Use whenever the user asks about "my Room Genie alerts", wants to create a new alert to watch pricing, check current rates, or manage existing alerts. Trigger on phrases like "watch this price", "create an alert", "my alerts", "any new availability", "Disney trip", "Disney vacation", "Disney cruise", "Disney hotel", "Room Genie".
---

# Room Genie — Core workflows

Room Genie monitors Disney hotel rooms and cruise staterooms for availability and price drops. This skill connects Claude to the user's Room Genie account via the `room-genie` MCP server.

## Tools available (from the `room-genie` MCP server)

| Tool | Purpose |
|---|---|
| `list_resorts` | Find a hotel resort (WDW / DLR / Aulani) by name or browse all |
| `list_room_types` | Get room UUIDs for a given resort — required before `create_alert` or `explore_rates` |
| `list_cruise_sailings` | Search Disney Cruise Line sailings by month, ship, nights |
| `list_stateroom_categories` | Get stateroom category codes for a specific sailing |
| `list_alerts` | List the user's alerts (filterable by active/paused and hotel/cruise) |
| `get_alert` | Fetch one alert with its most recent availability/pricing snapshot |
| `create_alert` | Create a new availability or price-drop alert (hotel OR cruise) |
| `toggle_alert` | Pause or resume an alert |
| `delete_alert` | Permanently delete an alert |
| `explore_rates` | Query live rates: `mode: "availability" | "room-only" | "package"` |

## Golden rules

1. **Always resolve IDs before pricing/creating.** Never pass a resort or room name as if it were an ID — call `list_resorts` / `list_room_types` (or `list_cruise_sailings` / `list_stateroom_categories` for DCL) first, then use the UUIDs you get back.

2. **Confirm before mutating.** `create_alert`, `toggle_alert`, `delete_alert` are destructive. For new alerts, recap the inputs (resort, dates, party, alert type) and ask the user to confirm before calling `create_alert`.

3. **Match the product to the user's phrasing.** If they say "Poly" or "Grand Floridian" → WDW hotel flow. "Disneyland Hotel" or "Grand Californian" → DLR. "Aulani" → Aulani (Aulani doesn't take tickets/dining/Memory Maker). "Wish", "Fantasy", "Treasure", "Destiny", "Adventure", "cruise" → DCL.

4. **Alert types:**
   - `availability` — fires when any room becomes available for sold-out dates. No target price needed.
   - `price_drop` — fires when the total price falls below `currentPrice`. User must supply a target.

5. **Party fields matter for pricing.** Always collect adults + children + each child's age before calling `explore_rates` in room-only or package mode.

6. **Ask about pricing shape before calling.** If the user hasn't explicitly said "package" or "room only", ask: *"Want a full package quote (tickets + dining + add-ons) or just room pricing?"* THEN use the product-specific skill (disney-world-planner / disneyland-planner / aulani-planner / disney-cruise-planner) to collect the right follow-ups for that product — the required fields and valid ranges differ per product.

7. **Never invent package fields.** If `mode: "package"`, you need: `ticketDays`, `ticketType`, `diningPlan`, `memoryMaker` (bool), `travelProtection` (bool). Valid ranges and defaults differ per product — ask.

8. **Multi-room requests in a single turn.** If the user asks for multiple rooms with different parties in the same message (e.g. "price a room for 2 adults and another for 2 adults + 1 child at the same resort"), call `explore_rates` once per party. Then present **one combined section**: per-room blocks with party/category/total/deposit/balance, plus a clearly labeled **Combined Package Total** line that sums the grand totals and a **Combined Deposit** line that sums the deposits. Same resort, same dates, just each room as its own card inside one response.

## Standard workflows

### "Create an alert for Poly Dec 1–5 for 2 adults, watch if it drops below $3000"

```
1. list_resorts({ query: "Poly" })  → get Polynesian resortId
2. list_room_types({ resortId })    → let user pick which room types to track (or use all)
3. Recap: "Watch Polynesian Resort, Deluxe Studio – Garden View, Dec 1–5, 2 adults,
           price_drop alert fires under $3000. Confirm?"
4. create_alert({ alert: { kind: "hotel", resortId, roomTypes: [...], checkIn: "2026-12-01",
                  checkOut: "2026-12-05", adults: 2, children: 0, childAges: [],
                  alertType: "price_drop", currentPrice: 3000, ... } })
```

### "What are my alerts?"

```
1. list_alerts({ activeOnly: false })
2. For each alert, render a row: name · dates · party · status · last checked
3. Offer follow-ups: toggle a paused alert, drill into one (get_alert), delete.
```

### "Check rates at Grand Floridian next weekend"

```
1. list_resorts({ query: "Grand Floridian" })
2. Get user's dates + party.
3. explore_rates({ mode: "availability" | "room-only", resortId, roomTypes: [],
                   checkIn, checkOut, adults, children, childAges })
4. Render the returned markdown table.
```

## Error handling

- **401 / auth errors** from MCP → tell the user their Claude client needs to reconnect to Room Genie (the OAuth session may have expired). Direct them to `/mcp` in Claude Code or the equivalent in Claude Desktop.
- **403 "Explorer subscription required"** → `mode: "room-only"` and `mode: "package"` require the $29 Explorer plan. Suggest they either use `mode: "availability"` (free across all tiers) or upgrade at https://app.roomgenie.travel/plans.
- **"No alert credits remaining"** on `create_alert` → direct them to https://app.roomgenie.travel/plans.

## Do NOT

- Do NOT call external APIs (Disney, booking sites) via WebFetch — the MCP server handles all Disney/DCL interaction server-side.
- Do NOT assume resort names map 1:1 to IDs — always look up.
- Do NOT guess stateroom category codes (like "O9C") — call `list_stateroom_categories` for the specific sailing.
- Do NOT fire `create_alert` without an explicit user confirmation.

## Suggest next steps after retrieving data

After any tool call, proactively offer 2–3 natural follow-ups as a short bulleted list. Match the suggestion to what you just returned — don't dump the full menu every time.

**After `list_alerts` returns results:**
- "Want me to show the latest availability for any of these? Just say the name."
- "Pause an alert?" / "Resume a paused one?" / "Delete one you're done with?"
- "Create another alert for a different resort or sailing?"

**After `get_alert` returns an alert with last-check data:**
- "Want me to re-check live rates right now?" (uses `explore_rates`)
- If the tracked rooms are sold out: "Check availability on nearby dates?" or "Try a different resort?"
- If there's an active offer: "Create a second alert at a lower target price to watch for a bigger drop?"

**After `explore_rates` in `availability` mode:**
- If anything is available: "Want a full price quote for any of these rooms?" (bump to `room-only` or `package`)
- If everything is sold out: "Want me to check +/- 2 days?" or "Check a different resort in the same tier?"
- **"Set an availability alert so you're notified if something opens up?"** — on yes, call `create_alert` with `alertType: "availability"`, `currentPrice: null`, and pull the resort/room/dates/party from the `structuredContent` of the result. Don't re-ask for anything already known.

**After `explore_rates` in `room-only` or `package` mode:**

First, present pricing clearly. ALWAYS include:
- **Party size** (e.g. "2 Adults, 1 Child age 8")
- **Resort name** (or sailing + ship name for cruise)
- **Room category / stateroom category** per room
- **Grand total** per room, big and upfront
- **Deposit due now** and **balance due** when the API returned them
- **Deposit due date** and **balance due date** when present (`paymentDates` in the structured result)
- **Offer name + discount** when an offer applies

If multiple rooms were returned for the same party: lay them out as comparable side-by-side blocks, cheapest first. If multiple parties (e.g. user asked "compare 2 adults vs 4 adults" and you called the tool twice), present BOTH blocks together and calculate the combined total if they said they'd book both.

THEN — and this is the primary next step — **offer to set a price-drop alert using the returned data**, one or two sentences max:

> "Want me to set a price-drop alert so you're notified if this drops below $X? I'll use the same resort, room, dates, and party from this quote."

Pick `X` as 5–10% below the cheapest grand total (round to nearest $50 for hotels, nearest $100 for packages). If the user says yes, do NOT re-ask for resort / dates / party / ticket config — pull all of it from the `structuredContent` of the `explore_rates` result you just returned, and call `create_alert({ alert: { kind: "hotel", resortId, roomTypes: [<room id you priced>], checkIn, checkOut, adults, children, childAges, ticketDays, ticketType, diningPlan, memoryMaker, travelProtection, alertType: "price_drop", currentPrice: <their target>, notificationMethod: "email", clientNote: "" } })`. Confirm the alert was created, then offer two secondary follow-ups:

- "Compare to a different resort for the same dates?"
- "Try shifting the dates by a week to see if midweek saves money?"
- If `mode: "package"`: "Try without Memory Maker or the dining plan to see the savings?"

For cruise (`sailingId` present): use `kind: "cruise"` and pull `stateroomCategory`, `stateroomType`, `shipName`, `sailingName` from the category the user showed interest in.

**After `list_resorts` or `list_room_types`:**
- "Which one do you want me to check for your dates?"
- Only ONE question — don't ask the user to pick AND give dates AND give party all at once. Drive one decision at a time.

**After `create_alert` succeeds:**
- "Want to check live rates right now so you know where we're starting?"
- "Create another alert for a different resort / sailing / date range as a backup?"
- "See all your active alerts?"

**After `toggle_alert` or `delete_alert`:**
- Keep it brief. One follow-up max: "Anything else to manage?" or move on silently if the flow is clearly ending.

Style: phrase suggestions as natural questions the user can answer yes/no or name, not as numbered menus. Keep it to **2–3 suggestions max** — more feels like interrogation.

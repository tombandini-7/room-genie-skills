---
name: room-genie-core
description: Use this skill EVERY TIME you are about to call any `mcp__room-genie-dev__*` or `mcp__room-genie__*` tool (list_alerts, get_alert, create_alert, toggle_alert, delete_alert, list_resorts, list_room_types, list_cruise_sailings, list_stateroom_categories, get_my_profile, explore_rates). Covers the required ordering of tool calls, confirmation rules before mutations, the post-result presentation format (per-room blocks with total/deposit/balance/due dates, multi-room combined totals), and the alert-offering flow (one alert per room, exact-price threshold suggestion, `get_my_profile` for notification defaults, availability alert for sold-out rooms). Applies across all four Room Genie products (WDW, DLR, Aulani, DCL). Also the right skill when the user mentions "Room Genie", "my alerts", "watch this price", "availability alert", or "price drop alert".
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
| `get_my_profile` | Read the caller's notification settings (email, SMS-ready?) — use before `create_alert` to pick sensible `notificationMethod` default |
| `list_alerts` | List the user's alerts (filterable by active/paused and hotel/cruise) |
| `get_alert` | Fetch one alert with its most recent availability/pricing snapshot |
| `create_alert` | Create a new availability or price-drop alert (hotel OR cruise). ONE per room — don't try to combine. |
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

6. **Dates come FIRST in any package conversation.** Some follow-up questions depend on the check-in year (e.g. WDW dining plans differ 2026 vs 2027+). If the user asks for a quote without dates, get dates + party first, then come back with product-specific package questions.

7. **Ask about pricing shape before calling.** If the user hasn't explicitly said "package" or "room only", ask: *"Want a full package quote (tickets + dining + add-ons) or just room pricing?"* THEN use the product-specific skill (disney-world-planner / disneyland-planner / aulani-planner / disney-cruise-planner) to collect the right follow-ups for that product — the required fields and valid ranges differ per product.

8. **Never invent package fields.** If `mode: "package"`, you need: `ticketDays`, `ticketType`, `diningPlan`, `memoryMaker` (bool), `travelProtection` (bool). Valid ranges, user-facing labels, and defaults differ per product — ask.

9. **User-facing labels ≠ API values.** When offering options to the user, use human-friendly names (e.g. "1 Park Per Day", "Park Hopper", "Waterpark & Sports", "Park Hopper Plus", "Disney Dining Plan", "Table Service"). Map to the API enum only when calling the tool. Never surface raw enum strings like `no-option` or `water-parks-sport` to the user.

10. **Memory Maker and Travel Protection are separate questions.** Always ask them as two distinct yes/no decisions, never bundled. They're different products with different pricing ($185 flat vs $99/adult).

8. **Multi-room requests in a single turn.** If the user asks for multiple rooms with different parties in the same message (e.g. "price a room for 2 adults and another for 2 adults + 1 child at the same resort"), call `explore_rates` once per party. Then present **one combined section**: per-room blocks with party/category/total/deposit/balance, plus a clearly labeled **Combined Package Total** line that sums the grand totals and a **Combined Deposit** line that sums the deposits. Same resort, same dates, just each room as its own card inside one response.

9. **LIST BEFORE YOU PRICE (hotels only) — HARD STOP.** For WDW, DLR, and Aulani, you MUST call `list_room_types({ resortId })`, present the room list to the user, and **wait for them to reply with their picks** before calling `explore_rates`. Do NOT price rooms on the user's behalf "to be helpful" — even if the user supplied dates, party, tickets, and every other field in their very first message. The rooms are the ONE thing the user always needs to choose. Each room is a separate 25–30s cart flow, so pricing all rooms at a Deluxe resort is ~5 minutes of scraping that the user almost never wants.

   The mandatory flow:

   ```
   list_resorts({ query })         → resortId
   list_room_types({ resortId })   → present the list in chat
   ⛔ STOP. Wait for the user to pick rooms. DO NOT CALL explore_rates YET.
   user replies with picks       → map names to ids
   explore_rates({ roomTypes: [only the picked ids] })
   ```

   Exception: only if the user explicitly typed "price every room" or "compare all rooms" in their message — then pass an empty `roomTypes` array and warn them about the duration. "Price me Grand Floridian" is NOT an explicit request for every room.

   For Disney Cruise Line this rule does NOT apply — see disney-cruise-planner for the cruise-specific flow (prices all categories in one scrape).

11. **Never invent a recipient, client, traveler, or reference name.** When presenting a quote, don't attach a human name the user didn't provide (e.g. don't write "Here's the Boettcher-style quote" or "For the Johnson family"). The header should reference the resort + dates + party, never a fabricated person. If the user gave a name, use it exactly; if they didn't, don't make one up.

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

If multiple rooms were returned for the same party: lay them out as comparable side-by-side blocks, cheapest first. If multiple parties (e.g. user asked "compare 2 adults vs 4 adults" and you called the tool twice), present BOTH blocks together. A **Combined Total / Combined Deposit** summary is fine for *display*, but alerts are set per-room — see below.

### Alert offer flow (RIGOROUS — follow exactly)

Alerts in Room Genie are **one per room**, not combined. If you quoted two rooms, that's two separate `create_alert` calls if the user wants both watched.

**Step A — figure out the notification method.**
Before proposing an alert, call `get_my_profile`. It returns `suggestedMethod`:
- If the user has a verified phone AND SMS consent, `suggestedMethod` will be `"both"` — propose `both` as the default.
- Otherwise `suggestedMethod` will be `"email"` — propose `email` only.
- NEVER propose `sms` or `both` if the user hasn't set up SMS. Instead say: *"I can email you. If you'd like text alerts too, set up SMS in Settings first."*

**Step B — propose the alert in a single concise offer.**
For each **available** room you quoted, offer one alert. Example for a single-room quote:

> "Want to watch this price? I'll set a price-drop alert at **$6,559.50** (today's exact price) — you'll be notified by **email** if it drops below that. Want to name it (optional) or keep the defaults?"

Key rules:
- **Suggest the EXACT grand total as the threshold**, not a discount. Let the user say "actually, make it $6,000" if they want a cushion.
- **Offer the name as optional.** If they don't supply one, pass `name: null` (or omit). Don't force a name.
- **Notification method comes from `get_my_profile`** — never guess.
- **For multi-room quotes:** list each room's proposed alert separately:
  > "If you want both watched, I can set:
  >  1. **Resort View — 2 adults** price-drop alert at $6,559.50
  >  2. **Resort View — 2 adults + child age 4** price-drop alert at $7,268.54
  >  Both by email. Want both, or just one?"

**Step C — on "yes", call `create_alert` per room.**
Do NOT re-ask resort / dates / party / ticket config — pull it from the `structuredContent` of the `explore_rates` result. One call per room the user wants watched:

```
create_alert({
  alert: {
    kind: "hotel",
    name: <user-provided name or null>,
    resortId, roomTypes: [<single room id>],
    checkIn, checkOut,
    adults, children, childAges,
    ticketDays, ticketType, diningPlan, memoryMaker, travelProtection,
    alertType: "price_drop",
    currentPrice: <grand total from this specific room's quote>,
    notificationMethod: <suggestedMethod from get_my_profile, or user's override>,
    clientNote: ""
  }
})
```

For cruise (`sailingId` present): use `kind: "cruise"` and pull `stateroomCategory`, `stateroomType`, `shipName`, `sailingName` from the category the user showed interest in.

### Sold-out rooms → offer an availability alert instead

For any room that came back `available: false` in the `explore_rates` result, you DON'T have a price to watch — a price-drop alert doesn't apply. Instead offer an **availability alert**:

> "The Theme Park View room is currently sold out for those dates. Want me to set an **availability alert** so you're notified the moment it opens up?"

On yes, call `create_alert` with `alertType: "availability"` and `currentPrice: null`. Everything else pulled from the quote's structured result.

### Secondary follow-ups (after alert offer)

Offer one or two of these, not all:
- "Compare to a different resort for the same dates?"
- "Try shifting the dates by a week to see if midweek saves money?"
- If `mode: "package"`: "Try without Memory Maker or the dining plan to see the savings?"

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

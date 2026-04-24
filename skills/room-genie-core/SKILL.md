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
| `show_price_matrix` | Render the full ticket + dining pricing grids from the cached last `explore_rates` package response. PRESENTATIONAL ONLY — no new scrape, no arguments needed. Call AFTER emitting the quote text to the user. |

## Golden rules

1. **Always resolve IDs before pricing/creating.** Never pass a resort or room name as if it were an ID — call `list_resorts` / `list_room_types` (or `list_cruise_sailings` / `list_stateroom_categories` for DCL) first, then use the UUIDs you get back.

2. **Confirm before mutating.** `create_alert`, `toggle_alert`, `delete_alert` are destructive. For new alerts, recap the inputs (resort, dates, party, alert type) and ask the user to confirm before calling `create_alert`.

   **Hotel alerts (WDW, DLR, Aulani) — ask package vs room-only first, same as `explore_rates`. This applies to BOTH `availability` AND `price_drop` alerts.** The alert will watch exactly what you pass — if you silently default a "Deluxe package" availability alert to room-only, the worker will check the wrong thing. Before calling `create_alert` with `kind: "hotel"`, ask the user:
   - *"Do you want to watch the room-only rate, or a full package?"*
   - If they pick package, ask the SAME product-specific follow-ups `explore_rates` asks. Use the EXACT user-facing labels below — never show API enum values, never invent "Base" or "Standard":
     - **WDW:** park days (2–10); ticket type — show these four labels exactly: **"1 Park Per Day"**, **"Park Hopper"**, **"Waterpark & Sports"**, **"Park Hopper Plus"**; dining plan — 2026 check-in: **"None"**, **"Quick Service"**, **"Disney Dining Plan"** (no Deluxe in 2026). 2027+ check-in: **"None"**, **"Quick Service"**, **"Table Service"**, **"Deluxe Table Service"**; Memory Maker y/n; Travel Protection y/n
     - **DLR:** park days (1–5); ticket type — show these two labels exactly: **"1 Park Per Day"**, **"Park Hopper"**; Travel Protection y/n (no dining, no Memory Maker)
     - **Aulani:** ONE yes/no question — **"Would you like Travel Protection added? (Yes / No)"**. Travel Protection is the ONLY Aulani add-on. Do NOT present a "package vs room-only" multi-choice at Aulani, and NEVER invent options labeled "Package with airfare", "Package with airfare/protection", "Air + hotel", "Ground transport", or "Flights" — none of those exist at Aulani through Room Genie. Aulani sells NO tickets, NO dining, NO Memory Maker, NO airfare, NO transport. If the user says yes to TP, pass `travelProtection: true` (and `userConfirmedChoices: true`); if no, pass `travelProtection: false, ticketDays: 0`
   - Then pass `userConfirmedChoices: true` along with every applicable add-on field explicitly (`ticketDays`, `ticketType`, `diningPlan`, `memoryMaker`, `travelProtection` — as applicable per product). For room-only, pass `ticketDays: 0, memoryMaker: false, travelProtection: false` with other fields left at defaults.
   - The server **refuses** hotel alert creation (both alert types) without `userConfirmedChoices: true` and returns a product-tailored follow-up list. Cruise alerts don't need this flag (cruise flow prices everything in one scrape).

   **ONE alert per room — call `create_alert` once PER ROOM, not once with a room array.** The `roomTypes` field on `create_alert` is always a single-element array. If the user wants to watch 3 rooms, make 3 separate `create_alert` calls (parallel is fine). Each call consumes its own credit (for non-subscribers) and creates its own row in the alert list. Package/room-only and naming choices can be gathered ONCE covering all rooms, but the create_alert call itself is one-per-room. This is different from `explore_rates`, which accepts a multi-room `roomTypes` array.

   **Notification method: leave unset, the server auto-picks.** Omit `notificationMethod` on `create_alert` and the server defaults it to `"both"` when the caller has a verified phone with SMS consent, otherwise `"email"`. Only pass it explicitly if the user asked for a specific channel. (Still call `get_my_profile` first when recapping so you can accurately say "by email and SMS" or "by email" in the confirmation prompt.)

3. **Match the product to the user's phrasing.** If they say "Poly" or "Grand Floridian" → WDW hotel flow. "Disneyland Hotel" or "Grand Californian" → DLR. "Aulani" → Aulani (Aulani doesn't take tickets/dining/Memory Maker). "Wish", "Fantasy", "Treasure", "Destiny", "Adventure", "cruise" → DCL.

4. **Alert types:**
   - `availability` — fires when any room becomes available for sold-out dates. No target price needed.
   - `price_drop` — fires when the total price falls below `currentPrice`. User must supply a target.

   **When the user asks for an `availability` alert, run a live check first.** Before calling `create_alert` with `alertType: "availability"`, offer to run `explore_rates` in `mode: "availability"` for the same resort/rooms/dates/party. This is fast and free (no Explorer subscription required, no credits consumed), and it tells the user whether the room is already available right now — in which case they may not need an alert at all, or they may want to book immediately.

   > "Before I set the availability alert, want me to run a quick live check? I can see right now whether any of those rooms are open for your dates — takes about 10 seconds."

   After the live check:
   - If at least one requested room **is available**: show the user what's open and ask: *"Want to grab it now, or still set the alert in case it sells out?"*
   - If everything is **sold out**: confirm back: *"Nothing's available right now — here's the alert I'll set up."* Then proceed with `create_alert`.
   - If the user declines the live check: skip straight to `create_alert`.

   This offer is specific to `availability` alerts. For `price_drop` alerts, the user has typically already seen a quote, so a second live check isn't useful.

5. **Party fields matter for pricing.** Always collect adults + children + each child's age before calling `explore_rates` in room-only or package mode.

6. **Dates come FIRST in any package conversation.** Some follow-up questions depend on the check-in year (e.g. WDW dining plans differ 2026 vs 2027+). If the user asks for a quote without dates, get dates + party first, then come back with product-specific package questions.

7. **Ask about pricing shape before calling.** If the user hasn't explicitly said "package" or "room only", ask: *"Want a full package quote (tickets + dining + add-ons) or just room pricing?"* THEN use the product-specific skill (disney-world-planner / disneyland-planner / aulani-planner / disney-cruise-planner) to collect the right follow-ups for that product — the required fields and valid ranges differ per product.

8. **Never invent package fields.** If `mode: "package"`, the required fields depend on the product. **WDW**: `ticketDays`, `ticketType`, `diningPlan`, `memoryMaker`, `travelProtection`. **DLR**: `ticketDays`, `ticketType`, `travelProtection` (NO `diningPlan`, NO `memoryMaker` — DLR doesn't support them). **Aulani**: `travelProtection` only (NO tickets/dining/MM). **DCL**: none of these apply — pass `sailingId`, `stateroomCategory`, `stateroomType`. Valid ranges, user-facing labels, and defaults differ per product — ask the product-specific skill.

9. **User-facing labels ≠ API values.** When offering options to the user — whether in chat or in Claude Desktop's AskUserQuestion / picker UI — use these EXACT labels. Never shorten, paraphrase, or invent names.

   **Ticket type labels (present as buttons/options):**
   - **"1 Park Per Day"** (NOT "Base", NOT "Standard", NOT "no-option")
   - **"Park Hopper"**
   - **"Waterpark & Sports"** (WDW only)
   - **"Park Hopper Plus"** (WDW only)

   **Dining plan labels — year-dependent (WDW only; DLR and Aulani don't have dining plans):**
   - **2026 check-in:** "None", "Quick Service", "Disney Dining Plan" — three options only (no Deluxe in 2026)
   - **2027+ check-in:** "None", "Quick Service", "Table Service", "Deluxe Table Service" — four options (NEVER "Standard", NEVER "Deluxe" alone)

   Map to the API enum only when calling the tool. Never surface raw enum strings (`no-option`, `water-parks-sport`, `standard`, etc.) to the user.

10. **Memory Maker is WDW-ONLY.** DO NOT ask users about Memory Maker for DLR, Aulani, or DCL — those products don't offer it as a package add-on. For WDW, ask Memory Maker and Travel Protection as two separate yes/no questions (different products, different pricing: $185 flat vs $99/adult). Never bundle them into a single question.

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

12. **ONE `explore_rates` call already priced every add-on variant — never re-call to swap an add-on, and never estimate.** A single WDW package response contains EXACT totals (already returned by Disney in the same API response) for every ticket-day × ticket-type combination, every dining plan, Memory Maker, and Travel Protection. They're surfaced in two places in the LAST `explore_rates` result:

    · **`structuredContent.addOnOptions`** — programmatic access:
      - `tickets.dayOptions[].typeOptions[].admissionTotal` — the FULL day × type grid. Exact admission total for ANY combination.
      - `tickets.typeOptions[].admissionDelta` — shortcut slice at the user's selected day count only.
      - `dining.planOptions[].diningCost` + `diningDelta`.
      - `memoryMaker.price`, `travelProtection.total`.
    · **"### Customize your package"** section of the text response — human-readable:
      - A pipe-table **"Full ticket pricing grid"** with every day × type cell priced.
      - Dining-plan deltas/totals, Memory Maker add cost, Travel Protection add cost.

    **Absolutely forbidden:**
    - Re-calling `explore_rates` just to swap a ticket type, day count, dining plan, or any add-on.
    - Estimating, rounding, or approximating — every number is in the response to the penny.
    - **Composing deltas** to fake a cross-combination. If the user originally chose 3-Day 1 Park Per Day and asks "what about 4-Day Park Hopper", you MUST read the `4 | Park Hopper` cell directly from the grid (or `dayOptions[where days=4].typeOptions[where id=park-hopper].admissionTotal`). Do NOT add a "4-day delta" to a "Park Hopper delta" — Disney's ticket pricing is non-linear (multi-day bulk pricing), so derived numbers are wrong. Always read the cell; never compose deltas.

    Only re-call `explore_rates` when the **resort**, **dates**, **party**, or **mode** (room-only vs package) changes.

13. **ALL ROOM QUOTES FIRST, THEN ONE `show_price_matrix` call at the END.** `explore_rates` returns the room quote(s). The full ticket pricing grid, the dining plan deltas, and the Memory Maker / Travel Protection add-on lines come from a separate tool called `show_price_matrix`, which reads the most recent explore_rates package response from a per-user server-side cache (10-min TTL). Do NOT batch-call both tools before writing any text — that pattern causes Claude to summarize the matrices as prose. Call `explore_rates` ONCE with ALL requested rooms in `roomTypes` (it prices every room in one scrape), emit every room's quote to the user FIRST, then call `show_price_matrix` with no arguments EXACTLY ONCE at the end. Do NOT try to pass `addOnOptions` — it lives in `structuredContent`, which is not visible to you. Do NOT call show_price_matrix per room — the matrix is identical across all rooms at the same resort+dates+party. Always call it before ending your turn; don't wait for the user to ask. The matrix is how the user sees what they're declining — especially the dining deltas when they picked "None".

    The flow after a successful hotel package explore_rates:

    ```
    1. Call explore_rates ONCE with every room the user wants in roomTypes.
    2. Emit EVERY room's quote text to the user as your reply (breakdown table, deposit, offer, per room).
    3. THEN call show_price_matrix()  — NO arguments, ONCE for the whole reply.
    4. Paste every block from show_price_matrix's response verbatim as the final section.
    5. Optional: add ONE short sentence afterward ("Want to go with any of these? Or should I set a price-drop alert?").
    ```

    Skip `show_price_matrix` only when:
    - explore_rates was `mode=availability` or `mode=room-only` (no addOnOptions)
    - The resort is Aulani (no tickets/dining/MM)
    - The call was for a cruise (sailingId)

    When show_price_matrix's content does arrive, paste it unchanged. Do NOT:
    - summarize ("here are a couple highlights…")
    - cherry-pick two rows and drop the rest
    - paraphrase cells into prose bullets ("Park Hopper would be +$223…")
    - collapse the tables into text
    - omit the preamble/postamble or the header rows

    The grids exist so the user can scan the full matrix at once — especially the dining deltas, which are the whole reason the user needs to see the matrix after declining dining.

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

**Step A — figure out the notification method (for the recap wording only).**
Before proposing an alert, call `get_my_profile`. It returns `suggestedMethod`:
- If the user has a verified phone AND SMS consent, `suggestedMethod` will be `"both"` — say "you'll be notified by email and SMS" in the recap.
- Otherwise `suggestedMethod` will be `"email"` — say "you'll be notified by email" in the recap.
- NEVER propose `sms` or `both` if the user hasn't set up SMS. Instead say: *"I can email you. If you'd like text alerts too, set up SMS in Settings first."*

You do NOT need to pass `notificationMethod` to `create_alert` — the server applies the same logic and defaults to `both` when SMS is ready, `email` otherwise. Only pass it explicitly if the user overrode the default (e.g. "email only, no texts").

**Step B — propose the alert in a single concise offer.**
For each **available** room you quoted, offer one alert. Example for a single-room quote:

> "Want to watch this price? I'll set a price-drop alert at **$6,559.50** (today's exact price) — you'll be notified by **email** if it drops below that."

Then ask about naming in a separate, explicit question — never skip it, never default silently:

> "How would you like to name this alert?
>  1. **I'll give it a name** — reply with the name you want.
>  2. **You pick a name for me** — I'll suggest a short descriptive one (e.g. "Poly Theme Park View — Dec 1–5").
>  3. **Leave it unnamed** — the alert list will show the resort + dates.
>
>  Which one?"

Key rules:
- **Suggest the EXACT grand total as the threshold**, not a discount. Let the user say "actually, make it $6,000" if they want a cushion.
- **ALWAYS ask about naming as an explicit three-choice question** — user-named, AI-named, or unnamed. Don't assume unnamed; don't invent a name without permission.
  - If the user picks **"user-named"**, wait for them to reply with the name, then pass it as `name: "<their string>"`.
  - If the user picks **"AI-named"**, generate a short descriptive label (≤50 chars) combining resort/room + dates, and pass it as `name: "<your suggestion>"`. Surface the name in your confirmation so they can override.
  - If the user picks **"unnamed"**, pass `name: null` (or omit the field).
- **Notification method comes from `get_my_profile`** — never guess.
- **For multi-room quotes:** list each room's proposed alert separately, then ask the naming question **once** with a follow-up clarifying whether the naming choice applies to all or each room individually:
  > "If you want both watched, I can set:
  >  1. **Resort View — 2 adults** price-drop alert at $6,559.50
  >  2. **Resort View — 2 adults + child age 4** price-drop alert at $7,268.54
  >  Both by email. Want both, or just one?"
  >
  > "For naming: (1) you name each, (2) I suggest names for each, or (3) leave them both unnamed?"

**Step C — on "yes", call `create_alert` once PER ROOM.**
Do NOT re-ask resort / dates / party / ticket config — pull it from the `structuredContent` of the `explore_rates` result. If the user wants 3 rooms watched, fire 3 separate `create_alert` calls (parallel is fine) — `roomTypes` always holds a single room id.

```
// Room 1
create_alert({
  alert: {
    kind: "hotel",
    name: <user-provided name or AI-suggested name or null>,
    nameChoice: "user_provided" | "ai_suggested" | "unnamed",  // REQUIRED — server rejects without this
    liveCheckAcknowledged: true,  // REQUIRED for alertType: "availability" — only after you offered the live check
    resortId, roomTypes: [<single room id>],
    checkIn, checkOut,
    adults, children, childAges,
    ticketDays, ticketType, diningPlan, memoryMaker, travelProtection,
    userConfirmedChoices: true,  // required for hotels; skipped for cruise
    alertType: "price_drop",  // or "availability"
    currentPrice: <grand total from this specific room's quote, or null for availability>,
    // notificationMethod: omit — server auto-picks both (SMS-ready) or email
    clientNote: ""
  }
})

// Room 2 — separate call, new roomTypes, may differ in currentPrice/name
// Room 3 — separate call, ... etc.
```

Never try to pass multiple room ids in one `create_alert` call. Never batch. One call, one room — always.

**Required fields that the server enforces on every `create_alert` call:**
- `nameChoice` — always. The server rejects the call if this is missing. Only set it after you've asked the three-choice naming question.
- `liveCheckAcknowledged: true` — only on `alertType: "availability"`. The server rejects availability alerts without this. Only set it after you've actually offered the live check (user declined or you ran it).
- `userConfirmedChoices: true` — only on `kind: "hotel"`. The server rejects hotel alerts without this. Only set it after you've asked package-vs-room-only and the product-specific add-on questions.

For cruise (`sailingId` present): use `kind: "cruise"` and pull `stateroomCategory`, `stateroomType`, `shipName`, `sailingName` from the category the user showed interest in.

### Sold-out rooms → offer an availability alert instead

For any room that came back `available: false` in the `explore_rates` result, you DON'T have a price to watch — a price-drop alert doesn't apply. Instead offer an **availability alert**:

> "The Theme Park View room is currently sold out for those dates. Want me to set an **availability alert** so you're notified the moment it opens up?"

On yes, gather the SAME package-vs-room-only questions as a price_drop alert (see rule 2): ask whether to watch room-only or the full package, and if package, collect the product-specific add-on fields with the exact labels. Then call `create_alert` with `alertType: "availability"`, `currentPrice: null`, `userConfirmedChoices: true`, and every applicable add-on field explicitly. Everything else (resort, room, dates, party) pulls from the quote's structured result. If the user wants multiple rooms watched, fire one `create_alert` per room.

### Secondary follow-ups (after alert offer)

Offer one or two of these, not all:
- "Compare to a different resort for the same dates?"
- "Try shifting the dates by a week to see if midweek saves money?"
- If `mode: "package"` at WDW: point to the "### Customize your package" section already in the response to show the exact savings from dropping Memory Maker, Travel Protection, or the dining plan — do NOT call `explore_rates` again to "check without X". (WDW-only — DLR/Aulani/DCL don't offer those add-ons.)

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

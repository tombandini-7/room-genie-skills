---
name: disney-world-planner
description: Use this skill BEFORE calling `explore_rates` or `create_alert` for a Walt Disney World resort (Polynesian, Grand Floridian, Contemporary, Wilderness Lodge, BoardWalk, Beach Club, Yacht Club, Caribbean Beach, Port Orleans Riverside/French Quarter, Pop Century, Art of Animation, All-Star Movies/Music/Sports, Saratoga Springs, Old Key West, Copper Creek, Riviera, Coronado Springs, Fort Wilderness, Swan, Dolphin, Swan Reserve, Animal Kingdom Lodge Jambo/Kidani) or any time the conversation is about a WDW trip in Florida — Magic Kingdom, EPCOT, Hollywood Studios, Animal Kingdom Park, Orlando. The skill contains the REQUIRED follow-up questionnaire (package vs room-only; if package: ticketDays 2-10, ticket type, dining plan, Memory Maker, Travel Protection), resort tier guidance (Value/Moderate/Deluxe/Deluxe Villa), and the post-quote suggestions. Do NOT guess package fields — use this skill to get the right questions.
---

# Walt Disney World — trip planner

Layer this on top of `room-genie-core`. Use the MCP tools from that skill; this skill adds WDW-specific domain knowledge for picking a resort and quoting full packages.

## Resort tiers (affects price and amenities)

| Tier | Examples | Typical nightly rate | Who it's for |
|---|---|---|---|
| Value | All-Star (Movies/Music/Sports), Pop Century, Art of Animation | $150–250 | Budget trips, larger families |
| Moderate | Port Orleans (Riverside/French Quarter), Caribbean Beach, Coronado Springs | $250–400 | Balance of price and theming |
| Deluxe | Contemporary, Grand Floridian, Polynesian, BoardWalk, Beach Club, Yacht Club, Wilderness Lodge, Animal Kingdom Lodge (Jambo/Kidani) | $500–900+ | Short walks/monorails to parks, best theming |
| Deluxe Villa (DVC) | Bay Lake Tower, Grand Floridian Villas, Polynesian Villas, BoardWalk Villas, Beach Club Villas, Old Key West, Saratoga Springs, Copper Creek, Riviera, Animal Kingdom Villas | $500–1500+ | Kitchens, more space, longer stays |
| Other | Swan, Dolphin, Swan Reserve (Marriott, Disney perks) | varies | Bonvoy points, often cheaper than Deluxe |

## Ticket types (`ticketType` for `explore_rates` / `create_alert`)

| Value | What it is |
|---|---|
| `no-option` | Standard one-park-per-day |
| `park-hopper` | Hop between parks same day |
| `water-parks-sport` | Park + Blizzard Beach/Typhoon Lagoon/sports admissions |
| `plus` | Park Hopper + Water Parks |
| `genie-plus` | Includes Lightning Lane (formerly Genie+) |
| `park-hopper-genie-plus` | Park Hopper + Lightning Lane |

Default to `no-option` if the user hasn't expressed a preference; ask before assuming hopper.

## Dining plans (`diningPlan`)

- `none` — no plan (most flexible)
- `quick_service` — counter-service only, cheapest plan
- `standard` — includes one table-service meal per night (the classic plan)
- `deluxe` — three table-service credits per night (foodies)

Don't push a plan unless the user asks. When they do ask "is the dining plan worth it?", steer them to run `explore_rates` twice (once with `none`, once with the plan they're considering) and compare total.

## Memory Maker + Travel Protection

- **Memory Maker**: flat $185 for unlimited PhotoPass photos across the trip. Worth it for 4+ day trips with photo-loving families.
- **Travel Protection**: $99 per adult, children free. Covers cancellation. Mention it for expensive trips.

## Required follow-up questions for WDW (ask before calling `explore_rates`)

Before you call `explore_rates` in `room-only` or `package` mode, confirm EVERY field below. Ask in a conversational bundle — one or two messages, not a form — but do NOT guess or default silently.

### Ask dates FIRST (they gate later options)

The year of the check-in date determines which dining plans Disney sells, so dates come first:

1. **Check-in and check-out dates** (YYYY-MM-DD).
2. **Party** — adults, number of children, and each child's age.
3. **Package or room-only?** If the user already said one, skip. Also skip if their phrasing makes it obvious ("full package", "with tickets", "just the room").

### If `package`, ask the rest — in this specific shape:

4. **Number of park days** (integer 2–10). WDW tickets start at 2 days.

5. **Ticket type** — present ONLY these four labels to the user, not the raw enum values:
   - **1 Park Per Day** (maps to `no-option`)
   - **Park Hopper** (maps to `park-hopper`)
   - **Waterpark & Sports** (maps to `water-parks-sport`)
   - **Park Hopper Plus** (maps to `plus`)

   Don't surface `genie-plus` / `park-hopper-genie-plus` as options in conversation — they exist in the API but aren't part of the standard consumer-facing pick list.

6. **Dining plan** — the offered set depends on the **check-in year**:

   - **2026 (check-in in 2026)** — offer:
     - None (maps to `none`)
     - Quick Service (maps to `quick_service`)
     - Disney Dining Plan (maps to `standard`)
   - **2027 and later** — offer:
     - None (maps to `none`)
     - Quick Service (maps to `quick_service`)
     - Table Service (maps to `standard`)
     - Deluxe Table Service (maps to `deluxe`)

   NEVER offer Deluxe for a 2026 check-in — Disney doesn't sell it that year. If the user requests Deluxe for a 2026 trip, tell them it only becomes available for 2027+ arrivals and fall back to Disney Dining Plan.

7. **Memory Maker?** — ask as a **separate** yes/no question. $185 flat, unlimited PhotoPass photos. Default no.

8. **Travel Protection?** — ask as a **separate** yes/no question. $99 per adult, children free. Default no, but mention for trips over ~$3k total.

Do NOT bundle Memory Maker and Travel Protection into the same question — they're distinct products with different pricing, and users decide on them separately.

### If dates are missing, do NOT ask packaging questions yet

If the user asks for pricing without giving check-in dates, get dates (and party) first. Saying "I'll need dates and party first to know which dining plans are offered that year" is fine. Once dates land, come back with the year-appropriate questionnaire.

Once you have all the answers, walk through the full-package workflow below.

## Full-package workflow

When the user says "quote me a full package at [resort]":

1. `list_resorts({ query })` → confirm resortId (disambiguate if multiple matches — Polynesian Resort vs Polynesian Villas).
2. `list_room_types({ resortId })` → show the user the choices, ask which view/bedding they want (or track all).
3. Collect the WDW follow-up answers above.
4. `explore_rates({ mode: "package", ... })` — this can take 2–4 minutes; warn the user.
5. Render the resulting table. Highlight: nightly rate, package grand total, deposit ($200 + travel protection), balance due, deposit due date, balance due date.
6. Offer to `create_alert` with `price_drop` if the user wants to be notified when the package drops below that price.

## Rendering hotel package output — STRICT rules

### Never round dollar amounts

Disney quotes WDW packages to the cent: e.g. **$8,033.69**, not $8,034. The tool emits the exact `\$X,XXX.XX` value with two decimals — render it as-is. Do NOT:

- Truncate to whole dollars ($8,034)
- Round to nearest 10 ($8,030)
- Drop trailing zeros ($8,033.7)
- Replace cents with "≈" or "~"

If you find yourself thinking "the cents are noise," they're not — users compare quotes against Disney's checkout page, which always shows the cent. A $0.69 mismatch makes the user distrust the entire quote.

### Per-room itemized breakdown (mirrors the web UI)

For each available room, the tool emits a fenced code block with the items the user opted into (room, tickets, dining, MM, TP) and a Grand Total. Example:

```
### Family Suites
_Suites_

```
Room (6 nights)              $4,172.37
4-Day 1 Park Per Day         $3,861.32
Table-Service Dining Plan    $1,773.30
─────────────────────────────────────
Grand Total                  $9,806.99
```

**Deposit:** $200.00 due Apr 21, 2026     **Balance:** $9,606.99 due Jan 15, 2027

Offer: **Standard Price**
```

Render this verbatim. Each line is one component the user selected — render every line, don't combine, don't summarize. The Grand Total at the bottom must equal the sum of the line items (and that's what the user verifies against Disney's checkout).

### Only show what the user opted into

The tool already filters: if the user said no Memory Maker, no MM line appears. If diningPlan is "none", no dining line. So if you don't see a line for something, it's because the user opted out — don't add it back, don't write a footnote saying "Memory Maker is also available for $185". Stay faithful to the breakdown the tool returned.

### One `explore_rates` call already priced every alternative — don't re-call

Disney's cart flow (the thing that takes 25–30 seconds per room) returns **every** ticket type, **every** day count, and **every** dining plan in the same response. The MCP tool passes all of it through in `structuredContent.addOnOptions`:

```
addOnOptions.tickets.typeOptions     ← admission total for 1 Park Per Day / Park Hopper / Waterpark & Sports / Park Hopper Plus, at the selected day count
addOnOptions.tickets.dayOptions      ← admission total for 2-day / 3-day / 4-day / … / 10-day, at each ticket type
addOnOptions.dining.planOptions      ← cost for No Plan / Quick Service / Standard / Deluxe
addOnOptions.memoryMaker.price       ← flat $185 add-on
addOnOptions.travelProtection.total  ← $99 × adults
```

When the user asks a follow-up like:
- "What would Park Hopper add?"
- "What if it's 5 days instead of 4?"
- "How much is the Deluxe dining plan?"
- "How much is Memory Maker?"

**Read from `structuredContent.addOnOptions` in the last `explore_rates` result. Do NOT call `explore_rates` again.** Re-calling launches a fresh browser + 25–30s cart flow for data you already have. That's the single biggest perf regression Claude can cause in this app.

Observed bug (v0.23 and earlier): Claude was calling `explore_rates` 5 times in parallel — once per ticket type — for the same room + dates. Each call ran a full cart flow. That's ~2.5 minutes of browser work to answer a question that was already answered by the first call.

### When it IS okay to re-call

Re-call `explore_rates` only when:
- Dates change
- Party size changes
- Resort changes
- User explicitly wants a fresh price (stale cache, over 2 minutes old)

Never re-call just to swap ticket type, ticket days, dining plan, Memory Maker, or Travel Protection on the same booking. Those are all in `addOnOptions`.

### Computing an alternative total from `addOnOptions`

If the user asks "what's the total with Park Hopper?", here's the math:

```
newTicketCost  = addOnOptions.tickets.typeOptions.find(t => t.id === 'park-hopper').admissionTotal
currentTicketCost = breakdown.ticketCost
newGrandTotal  = room.grandTotal - currentTicketCost + newTicketCost
```

Same pattern for dining (use `planOptions.find(p => p.dineType === '...').diningCost`). Memory Maker delta = `±185`. Travel Protection delta = `±(99 × adults)`. These are straightforward arithmetic on API-sourced numbers — acceptable to compute. Just say "from the same quote, swapping to Park Hopper would change the total to $X" so the user knows it's derived, not a fresh scrape.

### Don't fabricate alternatives not in addOnOptions

If the user asks about something not in `addOnOptions` (e.g. a different resort, different dates, different party), THEN a fresh `explore_rates` is correct. But pricing for the same room + dates + party + different add-ons? Always derive from the first call.

## WDW-specific gotchas

- **Animal Kingdom Lodge** is really two sub-resorts (Jambo House and Kidani Village) plus the villa side. `list_resorts` may return each separately — ask the user which one.
- **Park Hopper + Lightning Lane** (`park-hopper-genie-plus`) is often cheaper than buying them separately.
- **Free Dining promos** don't appear as a separate offer — if the user heard about one, tell them Disney's promotional engine returns the single best offer per room automatically via `explore_rates`, so no special flag is needed.
- **Swan / Dolphin / Swan Reserve** are non-Disney-owned. They work, but `explore_rates` uses a different cart flow under the hood and may be slower.

## Suggest next steps after retrieving WDW data

Tailor follow-ups to the resort tier and package shape. Two to three, never more.

**After a Deluxe / Deluxe Villa quote:**
- "Want me to compare against a Moderate for the same dates? Port Orleans Riverside and Caribbean Beach are usually the best value in that tier."
- "Try it as a DVC villa if you need more space — same resort area often has Studios within $50–100/night of the standard room."
- If ticketDays >= 5: "Want to see the same package with a Park Hopper added? It's usually a $80–100/person upgrade."

**After a Moderate quote:**
- "Want to see what it looks like if we drop to a Value (Pop Century or Art of Animation)? Usually saves $100–200/night."
- "Or push up to a Deluxe like Wilderness Lodge to see the tradeoff?"
- "Try it with the Dining Plan to see whether it breaks even for your party size?"

**After a Value quote:**
- "Want to see a Moderate for comparison? Sometimes seasonal deals close the gap."
- "Add tickets if you haven't already — need a park-day count?"

**After `explore_rates` finds nothing available:**
- "Try ±3 days? Disney's magical date-shuffling often finds rooms mid-week."
- "Pick a sibling resort in the same tier/area?" (suggest 1–2 specific names matching the one they tried)
- "Set an availability alert so we're notified if anything opens up?"

**After a full package total (with tickets + dining + MM + TP):**
- "Remove the Dining Plan and re-quote — see if the savings outweigh the convenience."
- "Try `travelProtection: false` — it's $99/adult. Worth removing if you already have travel insurance."
- "Want to set a price-drop alert at today's exact price? You can override if you want a buffer."

**After `list_room_types` returns many options:**
- Pick 2–3 likely matches (e.g., "Standard View" vs "Theme Park View" vs "Club Level") and describe the price/experience tradeoff in one line each. Don't dump all 15 rooms.

**After `create_alert` for WDW:**
- "Want to quickly check what's available at that resort right now so we know the baseline?"
- "Interested in setting a second alert at a cheaper Moderate as a backup?"

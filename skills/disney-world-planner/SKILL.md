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

## Ticket types

**Labels to show the user** (always quote these verbatim in chat and pickers):

- **"1 Park Per Day"**
- **"Park Hopper"**
- **"Waterpark & Sports"**
- **"Park Hopper Plus"**

Never show "Base", "Standard", or the API enum (`no-option`, `plus`, etc.) in user-facing text. Those are internal values Claude translates to on the tool call — not names users should ever see.

Default to "1 Park Per Day" if the user hasn't expressed a preference; ask before assuming hopper. `genie-plus` / `park-hopper-genie-plus` exist in the API but are not part of the consumer-facing pick list — do not offer them.

## Dining plans

**Labels to show the user** (year-dependent — check-in year determines the set):

- **2026 check-in** — offer exactly these three:
  - **"None"**
  - **"Quick Service"**
  - **"Disney Dining Plan"**
  - Never show "Deluxe" for 2026 — Disney doesn't sell it that year.
- **2027+ check-in** — offer exactly these four:
  - **"None"**
  - **"Quick Service"**
  - **"Table Service"**
  - **"Deluxe Table Service"**

Never show "Standard" — that's the API enum, not a user-facing label. Never show the enum values (`none`, `quick_service`, `standard`, `deluxe`) in chat.

Don't push a plan unless the user asks. When they do ask "is the dining plan worth it?", read the delta column from `show_price_matrix` — don't re-run `explore_rates`.

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

5. **Ticket type** — show the user EXACTLY these four labels, verbatim, and forbid the wrong forms by name in your own prompt:
   - **"1 Park Per Day"**
   - **"Park Hopper"**
   - **"Waterpark & Sports"**
   - **"Park Hopper Plus"**

   Never show "Base" (it is not a label we use) and never show the API enum (`no-option`, `park-hopper`, `water-parks-sport`, `plus`) in user-facing text. On the tool call itself, pass the enum value that corresponds to the label the user picked — but that translation happens silently.

6. **Dining plan** — the offered set depends on the **check-in year**. Present whichever full list applies:

   - **2026 check-in** — show exactly these three labels verbatim:
     - **"None"**
     - **"Quick Service"**
     - **"Disney Dining Plan"**

     Deluxe is NOT sold for a 2026 check-in — do not offer it. If the user requests Deluxe for a 2026 trip, tell them it only becomes available for 2027+ arrivals and fall back to Disney Dining Plan.

   - **2027+ check-in** — show exactly these four labels verbatim:
     - **"None"**
     - **"Quick Service"**
     - **"Table Service"**
     - **"Deluxe Table Service"**

   Never show "Standard" as a label — that is the API enum, not the user-facing name. Never show the bare enum values (`none`, `quick_service`, `standard`, `deluxe`) in chat.

7. **Memory Maker?** — ask as a **separate** yes/no question. $185 flat, unlimited PhotoPass photos. Default no.

8. **Travel Protection?** — ask as a **separate** yes/no question. $99 per adult, children free. Default no, but mention for trips over ~$3k total.

Do NOT bundle Memory Maker and Travel Protection into the same question — they're distinct products with different pricing, and users decide on them separately.

### Add-ons require tickets

Dining plans, Memory Maker, and Travel Protection are PACKAGE add-ons at WDW — Disney won't attach any of them without bundled tickets. If the user says they want, e.g., "no tickets but include Memory Maker", explain that the Memory Maker / dining plan / Travel Protection lines are sold only as part of a ticketed package, and ask them to either (a) include tickets (set `ticketDays >= 1` and pick a ticket type), or (b) drop the add-on(s) and run a `room-only` quote. The server will REFUSE a `mode="package"` call with `ticketDays = 0` plus any of those flags set — don't try to work around it.

### If dates are missing, do NOT ask packaging questions yet

If the user asks for pricing without giving check-in dates, get dates (and party) first. Saying "I'll need dates and party first to know which dining plans are offered that year" is fine. Once dates land, come back with the year-appropriate questionnaire.

Once you have all the answers, walk through the full-package workflow below.

## Full-package workflow

When the user says "quote me a full package at [resort]":

1. `list_resorts({ query })` → confirm resortId (disambiguate if multiple matches — Polynesian Resort vs Polynesian Villas).
2. **`list_room_types({ resortId })` — REQUIRED before pricing.** Show the user the full list of rooms at that resort and ask which 1–4 they want priced. Never skip this step and never default to "price everything" without explicit user consent — each room is a separate 25–30s cart flow.
3. User picks rooms → record the `roomTypes` UUID array.
4. Collect the WDW follow-up answers above (dates, party, package vs room-only, ticket days / type, dining plan, Memory Maker, Travel Protection).
5. `explore_rates({ mode: "package", resortId, roomTypes: [<picked ids>], ... })` — this takes 25–30 seconds per room. Warn the user about expected duration (e.g. "pricing 3 rooms will take about 90 seconds").
6. Render the resulting table. Highlight: nightly rate, package grand total, deposit ($200 + travel protection), balance due, deposit due date, balance due date.
7. Offer to `create_alert` with `price_drop` if the user wants to be notified when the package drops below that price.

### How to present the room list (step 2)

Use `list_room_types`'s output as-is. It returns id + name + category + view + bed type + max occupancy. Render as a short markdown list or table with the info, then ask something like:

> "Which rooms would you like me to price? You can pick several (e.g. 'Standard View and Theme Park View'). Pricing takes about 25–30 seconds per room."

Accept partial names like "Standard View" — match them to the UUIDs in the tool's output. If the user says "all of them", confirm first: "That's N rooms, about M minutes of pricing. Are you sure?"

## Rendering hotel package output — STRICT rules

### Never round dollar amounts

Disney quotes WDW packages to the cent: e.g. **$8,033.69**, not $8,034. The tool emits the exact `\$X,XXX.XX` value with two decimals — render it as-is. Do NOT:

- Truncate to whole dollars ($8,034)
- Round to nearest 10 ($8,030)
- Drop trailing zeros ($8,033.7)
- Replace cents with "≈" or "~"

If you find yourself thinking "the cents are noise," they're not — users compare quotes against Disney's checkout page, which always shows the cent. A $0.69 mismatch makes the user distrust the entire quote.

### Per-room itemized breakdown (mirrors the web UI)

For each available room, the tool emits a **markdown pipe table** (`| Item | Amount |` rows with a `| --- | ---: |` separator) with the items the user opted into (room, tickets, dining, MM, TP) and a Grand Total. Example shape (the tool's actual output — paste it through unchanged):

```markdown
### Family Suites
_Suites_

| Item | Amount |
| --- | ---: |
| Room (6 nights) | \$4,172.37 |
| 4-Day 1 Park Per Day | \$3,861.32 |
| Table-Service Dining Plan | \$1,773.30 |
| **Grand Total** | **\$9,806.99** |

**Deposit:** \$200.00 due Apr 21, 2026     **Balance:** \$9,606.99 due Jan 15, 2027

Offer: **Standard Price**
```

Render this verbatim — every `|`, every header, every separator row. Do **NOT** convert it into a fenced code block (` ``` `), do NOT realign cells with spaces or tabs, do NOT collapse it to bullets. The user's renderer (Claude Code, Claude Desktop, web) only draws a styled table when the pipe-table source survives. Each row is one component the user selected — render every row, don't combine, don't summarize. The Grand Total at the bottom must equal the sum of the line items (and that's what the user verifies against Disney's checkout).

### Only show what the user opted into

The tool already filters: if the user said no Memory Maker, no MM line appears. If diningPlan is "none", no dining line. So if you don't see a line for something, it's because the user opted out — don't add it back, don't write a footnote saying "Memory Maker is also available for $185". Stay faithful to the breakdown the tool returned.

### One `explore_rates` call already priced every alternative — use `show_price_matrix`, don't re-call

Disney's cart flow (the thing that takes 25–30 seconds per room) returns **every** ticket type, **every** day count, and **every** dining plan in the same response. The MCP server caches that payload for 15 minutes so a companion tool, `show_price_matrix`, can render it as two markdown tables on demand:

- The full **day × ticket-type admission grid** (every day count 2–10, crossed with every ticket type, with exact admission totals).
- The full **dining-plan table** — every plan's total and "Δ vs current" cost relative to the user's pick.

**How to use it.** After a WDW package `explore_rates` call:

1. Emit EVERY room quote as your reply first (each room's breakdown table, deposit, offer, etc.).
2. Then call `show_price_matrix` once, with no arguments — the server reads the cache automatically. Do not try to pass `addOnOptions`; Claude cannot see `structuredContent` at all anymore.
3. Paste the matrix output verbatim as the final section of the same reply.

Never call `show_price_matrix` between rooms or once per room — one call at the END of the reply covers every room (the matrix is identical across rooms at the same resort/dates/party).

When the user asks a follow-up like:
- "What would Park Hopper add?"
- "What if it's 5 days instead of 4?"
- "How much is Deluxe Table Service?"
- "How much is Memory Maker?"
- **"Give me dining plan options as the incremental cost"** / "show me dining upsells" / "what's each dining plan add?"

**Read from the `show_price_matrix` output you already pasted. Do NOT call `explore_rates` again.** Re-calling launches a fresh 25–30s cart flow for data already on screen. That's the single biggest perf regression Claude can cause in this app. If the matrix is stale (more than 15 minutes since the last explore_rates call, or the user changed dates/party/rooms), you'll need a fresh `explore_rates` — `show_price_matrix` will say "no recent quote" and you should start over from step 1.

### Dining upsell — the canonical case

If the user asks for "dining plan options as incremental costs" (common for travel-agent-style upsell prep), ONE `explore_rates` call with the user's preferred dining pick (usually "None") is all you need, followed by a single `show_price_matrix` call. The dining table in the matrix shows every plan's total and Δ — read directly from it; never estimate.

Observed bug (v0.23 and earlier): Claude was calling `explore_rates` 5 times in parallel — once per ticket type — for the same room + dates. Each call ran a full cart flow. That's ~2.5 minutes of browser work to answer a question that was already answered by the first call + one matrix render.

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
- Close with the THREE-WAY POST-MATRIX PROMPT from `room-genie-core`: "(a) price-drop alert on a room, (b) availability alert on any sold-out room (skip if everything is available), or (c) build a branded PDF quote for the client (calls `generate_quote_pdf` — gather inputs first)."
- Don't pile on extra suggestions; that prompt is the canonical close.

**After `list_room_types` returns many options:**
- Pick 2–3 likely matches (e.g., "Standard View" vs "Theme Park View" vs "Club Level") and describe the price/experience tradeoff in one line each. Don't dump all 15 rooms.

**After `create_alert` for WDW:**
- "Want to quickly check what's available at that resort right now so we know the baseline?"
- "Interested in setting a second alert at a cheaper Moderate as a backup?"

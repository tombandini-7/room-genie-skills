---
name: disney-cruise-planner
description: Use this skill BEFORE calling `list_cruise_sailings`, `list_stateroom_categories`, `explore_rates` (with sailingId), or `create_alert` (kind:"cruise") for any Disney Cruise Line trip — Disney Wish, Disney Fantasy, Disney Treasure, Disney Dream, Disney Magic, Disney Wonder, Disney Destiny, Disney Adventure — or any DCL itinerary (Bahamian, Caribbean, Alaskan, Mediterranean, Transatlantic, Castaway Cay, Lookout Cay, Port Canaveral). Cruise flow is STRUCTURALLY DIFFERENT from hotels: no ticket days, no dining plan toggle, no Memory Maker — the cruise fare already bundles those. Contains the DCL questionnaire (months array for search, party, optional ship/nights; then sailingId + category + mode for pricing), stateroom category structure (Inside/Oceanview/Verandah/Concierge), ship codes (WW=Wish, WD=Destiny, DF=Fantasy, DA=Adventure, DD=Dream, DT=Treasure, DM=Magic, DW=Wonder), and kid-pricing nuances.
---

# Disney Cruise Line — trip planner

Layer this on top of `room-genie-core`. DCL is structurally different from hotels: you book a *sailing* (a specific ship + departure date) and pick a *stateroom category* (a room type code like "O9C").

## The ships

| Code | Ship | Typical routes |
|---|---|---|
| `WD` | Disney Destiny | (New 2025) — Caribbean, Bahamas |
| `WW` | Disney Wish | Bahamas, Caribbean |
| `DF` | Disney Fantasy | 7-night Caribbean |
| `DA` | Disney Adventure | (New) — Asia-Pacific |
| `DD` | Disney Dream | Europe |
| `DT` | Disney Treasure | Caribbean |
| `DM` | Disney Magic | Europe, transatlantic |
| `DW` | Disney Wonder | Alaska, Mexico, Pacific |

Pass the 2-letter code as `ships` filter on `list_cruise_sailings` when the user wants a specific ship.

## Stateroom category shape (4 broad types)

| Type | Letter prefix | Examples | Typical vibe |
|---|---|---|---|
| Inside | `I`, `11` | `11C`, `11B`, `11A` | No window, cheapest |
| Oceanview | `O`, `8`, `9` | `O9C`, `8D`, `9B` | Porthole or window, no verandah |
| Verandah | `V`, `4`, `5`, `6`, `7` | `V7A`, `5C`, `4A` | Private balcony |
| Concierge | `T`, `V1`, `V2`, `V3` | `V3B`, `T`, `R` | Concierge lounge access, top decks, pricier |

The exact category codes come from `list_stateroom_categories` — **never guess**; always call the tool to get the current list for a specific sailing.

## Required follow-up questions for DCL (ask before calling `list_cruise_sailings` or `explore_rates`)

DCL questionnaire is different from hotels. Before searching, confirm:

**For sailing search (`list_cruise_sailings`):**
1. **Month(s)** — `YYYY-MM` format, at least one. Ask "what month(s) are you flexible on?".
2. **Party** — adults, number of children, each child's age (ages matter for pricing; under-3s are discounted).
3. **Ship preference** — STRONGLY RECOMMEND picking at least one. Full-month-all-ships searches take 2+ minutes; narrowing to a ship or two brings it to 20–40 seconds. Present the list: Wish (WW), Fantasy (DF), Treasure (DT), Destiny (WD), Dream (DD), Magic (DM), Wonder (DW), Adventure (DA). If the user says "any ship", warn them the search will take 1–2 minutes and proceed.
4. *(Optional)* **Nights** — 3, 4, 5, 7+.

**Search performance note:** `list_cruise_sailings` scrapes DCL live. Scope directly drives latency:
- One specific ship + one month → ~20 seconds
- Two or three ships + one month → ~40–60 seconds
- All ships + one month → ~120 seconds (close to the MCP tool-call limit)
- Multiple months without ship filter → often times out; split into per-month calls if needed

**When the user asks for a specific target date (e.g. "closest to April 8"):**
The tool returns every matching sailing for the month filters, which can be 30–50 rows for a popular ship. Don't dump all of them. Instead:

1. Pick the tightest relevant month filter — one month is almost always enough; add the previous or next month only if the target date is within a week of the month boundary.
2. After the call returns, sort results by `|departureDate - targetDate|` in JavaScript (mentally) and show only the **3–5 closest** sailings — nearest match first, then nearest alternatives on either side.
3. Say something like: *"Closest sailings to April 8, 2027: [list]. Want the full month of options, or pick one of these to price?"*

This also dodges the overfetch problem where Render returns 51 sailings for a full month but the user only cares about 2–3 of them.

**For pricing a specific sailing (`explore_rates` with `sailingId`):**
1. **Sailing ID** — from the search result.
2. **Stateroom preference** — Inside (cheapest), Oceanview, Verandah, or Concierge (pricey, limited). Ask which category fits their budget.
3. **Mode:**
   - `availability` — "is the category open, how many left" — free.
   - `package` — full pricing with tax. Requires Explorer.
4. **Placeholder discount?** — REQUIRED ASK before pricing. Phrase it like:
   > *"Are you booking this with a Disney Cruise Line **placeholder** (also called Booking a Future Cruise Onboard)? It gets you 10% off the voyage fare and a $250-reduced deposit since you prepaid that on your prior cruise. Or is this your first DCL cruise / no placeholder?"*
   
   - If yes → you **MUST** call `explore_rates({ ..., placeholder: true })`. The placeholder flag is what triggers the tool to compute the adjusted prices.
   - If no → call without `placeholder` (or `placeholder: false`) so the user sees Disney's standard pricing.
   - Do not skip this question — it changes the numbers materially.
   
   ### Wait for the answer before calling `explore_rates`. Don't speculatively fetch.
   
   Claude's instinct is to be helpful by saying *"while you answer, I'll start fetching"* and immediately calling `explore_rates`. **Don't.** The placeholder flag is part of the tool input — calling without it means the prices are wrong, and you'll have to recall the tool (each call is 30–120s). That's a worse experience than waiting 5 seconds for the user's reply.
   
   What IS safe to do in parallel with the placeholder question:
   - `list_cruise_sailings` (only if you don't already have the sailingId) — it doesn't take a `placeholder` param.
   - `get_my_profile` — for notification setup later.
   
   What is NOT safe to do until the user answers:
   - `explore_rates({ sailingId, ... })` — placeholder is part of the input.
   - Any pricing-presentation work (no point until you have prices).
   
   When you ask the question, say *"I'll wait on that before pulling pricing — meanwhile I'm finding the sailing"* (if running list_cruise_sailings in parallel) or just *"Let me know and I'll pull the pricing."* (if you already have the sailingId). Never say *"While you answer that, I'll fetch the pricing"* — that's the wrong call.

   ### Deposit-due timing differs by mode

   - **Without placeholder:** Disney's standard window applies — deposit is typically due in a few days (Disney returns the exact date in the API). The tool shows the date Disney returned.
   - **With placeholder:** the placeholder deposit is due **immediately** on the day the booking is converted from the placeholder. There's no multi-day grace period — DCL requires payment same-day. The tool now labels the placeholder deposit row "due immediately"; surface that to the user when they ask about timing.

   ### How placeholder pricing actually works (read this carefully — Claude has gotten it wrong)
   
   **Disney's APIs do NOT return placeholder pricing.** Disney has no idea which user holds a placeholder; they always return standard fare/tax/total. The placeholder math is computed by the tool inside `format.ts` using Disney's exact pricing as the input. The formula is:
   
   ```
   netFare        = total - tax
   discount       = netFare × 0.10
   adjustedTotal  = total - discount
   adjustedFare   = adjustedTotal - tax
   standardDep    = adjustedFare × 0.10
   placeholderDep = max(0, standardDep - 250)
   ```
   
   When you call `explore_rates` with `placeholder: true`, the tool's text output **already contains** the adjusted values. The summary table headers swap to `Total (10% off)` and `Placeholder Deposit`, and each detail block adds `10% off fare`, `Adjusted total`, `Adjusted fare`, `Standard deposit`, and `Placeholder deposit` rows. **Render those rows as-is.**
   
   ### NEVER write any of these (Claude has been caught doing this):
   
   - "The server returned null for placeholder-adjusted pricing"
   - "The tool was unable to apply the 10% fare discount automatically"
   - "Disney's API didn't return placeholder data"
   - "Confirm the exact numbers with DCL or a travel agent before booking" (specific to placeholder — Disney won't quote placeholder prices)
   - Any other note implying the placeholder math was attempted server-side and failed
   
   These are wrong. Disney never returns placeholder pricing, by design. The math happens in the tool. If the placeholder rows aren't in your output, it's because you forgot to pass `placeholder: true` — re-call the tool with the flag set. Don't write an excuse.

5. **Do NOT ask:** ticket days, dining plan, Memory Maker, travel protection — DCL doesn't offer these as toggles. The cruise fare already includes onboard dining (rotational dining + Cabanas + QSRs). Gratuities are extra but not configurable.

**If the user wants a price-drop alert:**
- Ask for a specific category code (e.g. "O9C") by running `list_stateroom_categories` first — alerts track one category.

## Workflow — "what Disney cruises are available in Sep 2026?"

1. `list_cruise_sailings({ months: ["2026-09"], adults: 2, children: 0, childAges: [] })`
   - Optional filters: `ships`, `nights`, `destinations`, `ports`
2. Render the sailing list: sailing ID, ship, itinerary, dates, nights, availability.
3. Ask: "Which sailing looks interesting?" → grab the sailingId.
4. `list_stateroom_categories({ sailingId, adults, children, childAges })` → show category pricing + room counts.
5. User picks a category → `category` field (e.g. `O9C`) + often a human-friendly `stateroomType` (e.g. `OUTSIDE-DELUXE`).
6. Offer to alert or price.

## Workflow — "alert me if the Wish 4-nighter in Sep drops below $3200"

1. Find the sailing via `list_cruise_sailings`.
2. Find the category via `list_stateroom_categories` → get `category`, `stateroomType`.
3. Recap: "Watching Disney Wish 4-Night Bahamian Sep 7–11, 2026, Category O9C (Deluxe Oceanview Stateroom), 2 adults. Price-drop alert fires under $3200. Confirm?"
4. `create_alert({ alert: { kind: "cruise", sailingId, stateroomCategory: "O9C", stateroomType: "OUTSIDE-DELUXE", shipName: "Disney Wish", sailingName: "4-Night Bahamian Cruise", checkIn: "2026-09-07", checkOut: "2026-09-11", adults: 2, children: 0, childAges: [], alertType: "price_drop", currentPrice: 3200, notificationMethod: "email", clientNote: "" } })`

## Exploring rates at a cruise level

`explore_rates({ mode, sailingId, checkIn, checkOut, adults, children, childAges })` — two modes:
- `availability` — lightweight check: which categories are open and how many rooms left.
- `package` — full cruise pricing across categories. The cruise itself is the "package" at DCL; there are no separate tickets or dining plans.

`room-only` is not meaningful for DCL; if the user asks for it, use `availability` instead.

## DCL-specific gotchas

- **Sailing IDs** look like `WW0509` (Wish, sailing 0509). Don't invent them — always pull from `list_cruise_sailings`.
- **Child ages affect pricing** — kids under 3 sail at a discounted rate; collect ages explicitly.
- **Concierge cabins are very limited** — `roomCount` is often 2–5 for Concierge categories. Sellout risk is real; an availability alert is valuable.
- **Port Canaveral embarkation** is the default for the Florida-based ships (Wish, Fantasy, Dream, Treasure, Destiny). European/Alaskan sailings use regional ports (Barcelona, Vancouver, etc.).
- **Special offers** (Kids Sail Free, military rates, Florida resident) show up in `list_cruise_sailings` response when applicable. Surface them when present.
- **Gratuities are not included** — typically $14.50 per guest per night. Don't promise "all-inclusive."

## Rendering `list_stateroom_categories` / `explore_rates` (cruise) output

**Render the tool's `content[0].text` verbatim.** Do NOT re-format it into a tighter summary table of your own. The tool already emits a hybrid layout per group: a compact **summary table** and a **detailed block per category**. Keep BOTH. Keep EVERY column. Keep EVERY row.

### The output has two surfaces per group — render both EXACTLY as-is.

1. A **proper 7-column markdown pipe table** (using `|` separators, not a fenced code block). Columns: **Cat, Name, Total, Tax, Deposit, Gratuities, Rooms**. Markdown renderers display this as an actual bordered table — that's intentional; the user specifically wants a real table, not fenced plain text.
2. An **H3 heading** (`### <Cat> — <Name>`) followed by a **fenced code block** for each category with aligned key: value rows. Fields: Total, Fare, Tax, Gratuities, Out-the-door, Deposit, Final payment, Availability, Size, Sleeps, View, Decks, Location.

DO NOT:

- Modify the markdown pipe table (don't drop columns, don't rename headers, don't collapse to ASCII, don't remove column separators)
- Strip the H3 heading or the code fences around the detail block
- Flatten the key: value fenced block into single `·`-separated lines
- Write your own intro summary of gratuities before the table
- Add `~` to any price

Claude has been caught (a) compressing the table to 4–5 columns, (b) flattening the detail blocks into single lines, and (c) converting the H3 heading to plain text prefixed with "Category:". Every previous attempt to fix via skill-only directives failed; the table + fenced-detail structure is now enforced in the format code. Pass it through exactly.

### No Claude-synthesized preamble or postscript about gratuities

Do NOT write intro text like "Gratuities auto-apply at Disney's default rate: $80/guest standard, $136.25/guest concierge (sailing total, ~$16/night and ~$27.25/night)" before the table. The per-category Gratuities column and the detail block's Gratuities row already carry that information with the EXACT rate for the sailing. Inventing a summary preamble introduces:

- Rounding/tildes that contradict the exact numbers below
- Rate figures that Claude estimates rather than reads

If the user asks specifically for the gratuity rate, cite the value from the tool's structuredContent (`gratuitiesRate` or `conciergeGratuitiesRate`) — never a computed average.

### Every price in the output is EXACT. NEVER prefix with `~`.

- Totals, fares, taxes, deposits, final payments, gratuities — all come from Disney's APIs (availability + payment-calculator + sailing-details enrichment)
- `gratuitiesRate` and `conciergeGratuitiesRate` in the tool's structured content are Disney's actual per-sailing rates, not rounded averages
- **Never write `~$X,XXX` or `approximately $X,XXX` or `about $X,XXX`** when rendering a number from the tool. They are exact figures for this sailing + party + category combination
- The only exception: if Disney genuinely returned null for a field and the tool noted the fallback, surface that as `fallback: not returned by Disney` — never invent a tilde

**Special note on gratuities.** DCL auto-applies gratuities to every booking at the rate returned by Disney's sailing-details API. Removing gratuities is an explicit opt-out the user has to make at checkout — it's not optional-by-default. So the tool emits a line labeled "Out-the-door (fare + tax + default gratuities): $X" — render it as-is. Do NOT:
- Rename it to "With gratuities (~$X)"
- Add a "~" prefix to the number
- Replace "Out-the-door" with "With gratuities" or "Approximately"
- Put gratuity amounts in parentheses as if they were optional

Same rule for the "X rooms available · Y sq ft" line — no tildes on sq ft. Disney's stateroom-details API returns exact square footage.

### NEVER recompute Disney's deposit, tax, gratuity, or final payment amounts

Those numbers come from Disney's own payment-calculator API via the tool and are already in the text output (and in `structuredContent.categories[].deposit` / `.price.tax` / the `gratuitiesRate` field). Disney's deposit, gratuity, and final-payment formulas vary per sailing, per category, and per concierge tier — there is no single percentage you can apply. Pass through whatever the tool emitted. Full stop.

Each detailed block from the tool contains — and ALL of this must reach the user:

- **Category code** (e.g. `11C`)
- **Display name** (e.g. "Standard Inside Stateroom")
- **Total price** (tax-inclusive)
- **Fare + Tax split** — both lines matter, show them
- **With gratuities** line — the true out-the-door number (fare + tax + gratuities)
- **Deposit amount + due date**
- **Final payment due date**
- **Rooms available** count
- **Square footage** when present
- **Sleeps** range (e.g. "Sleeps 3–4")
- **View type** (e.g. "No exterior view", "Large porthole window", "Private Verandah")
- **Decks** the category is found on (e.g. "Decks 6, 7, 8")
- **Positioning** (e.g. "Midship", "Forward, Aft")

If you absolutely must summarize (e.g. the user explicitly asks "just show me the cheapest per group"), confirm the trim with a sentence first — *"I'll collapse the full category list into just the cheapest per group. Want the full list instead?"* Default behavior is full rendering.

## Suggest next steps after retrieving DCL data

**After `list_cruise_sailings` returns multiple sailings:**
- "Want me to check stateroom pricing on any of these? Pick a sailing ID."
- If many similar options: "The [ship] on [date] is usually the sweet spot for price — want me to dig into that one first?"
- If the user hasn't picked a ship yet: "Want to narrow to a specific ship? The Wish and Treasure are the newest and usually priciest."

**After `list_stateroom_categories` returns pricing for a sailing:**
- "Inside categories are cheapest — $X total here. Want to see what the jump to a Verandah costs? Usually $500–1500 more for a private balcony."
- If a Concierge category is still available: "Only [N] Concierge cabins left at this price — want to set an availability alert in case you need to decide quickly?"
- "Want a price-drop alert on a specific category, or the whole sailing?"

**After `explore_rates` in `availability` mode for a cruise:**
- Highlight 1–2 categories that match the user's budget (Inside for budget, Verandah for mid, Concierge for premium).
- "Want full pricing on those? I'll run `mode: "package"` for the categories you care about."
- If the target category is sold out: "Want to widen to any Verandah (or any Oceanview) instead of the specific code?"

**After a full cruise price quote:**
- "Add ~$14.50/guest/night for gratuities — want me to calculate the actual out-the-door total?"
- "Want to check a nearby sail date? Shifting by a week or swapping ships often saves meaningfully."
- "Want a price-drop alert at today's exact total ($X)? You can override the threshold if you want a cushion."

**After the user seems stuck between two ships:**
- "Wish vs Fantasy depends on itinerary — the Wish runs shorter Bahamian/Caribbean, Fantasy does 7-night Caribbean. Want me to compare a specific matching sail pair?"

**When the party includes young kids (under 3):**
- "Note your [N] under-3s sail at the youth rate — the per-person total you see already reflects that."
- "If your youngest is close to 3 on the sail date, shifting earlier can save real money — worth checking?"

**After `create_alert` for a cruise:**
- "Concierge inventory moves fast on DCL — want a second alert at a cheaper Verandah as a fallback?"
- "Want to check other sail dates on the same ship to compare pricing?"

**When the user mentions they're flexible on dates:**
- "Disney Cruise pricing has strong seasonal bands — January and mid-September are the cheapest weeks. Want me to check those?"

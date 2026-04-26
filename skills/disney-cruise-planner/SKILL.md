---
name: disney-cruise-planner
description: Use this skill BEFORE calling `cruise_search_sailings`, `cruise_list_categories`, or `create_alert` (kind:"cruise") for any Disney Cruise Line trip — Disney Wish, Disney Fantasy, Disney Treasure, Disney Dream, Disney Magic, Disney Wonder, Disney Destiny, Disney Adventure — or any DCL itinerary (Bahamian, Caribbean, Alaskan, Mediterranean, Transatlantic, Castaway Cay, Lookout Cay, Port Canaveral). Cruise flow is STRUCTURALLY DIFFERENT from hotels: no ticket days, no dining plan toggle, no Memory Maker — the cruise fare already bundles those. Contains the two-step DCL flow (cruise_search_sailings → cruise_list_categories), the questionnaire (months array for search, party, optional ship/nights; then sailingId + placeholder yes/no), stateroom category structure (Inside/Oceanview/Verandah/Concierge), ship codes (WW=Wish, WD=Destiny, DF=Fantasy, DA=Adventure, DD=Dream, DT=Treasure, DM=Magic, DW=Wonder), and kid-pricing nuances.
---

# Disney Cruise Line — trip planner

Layer this on top of `room-genie-core`. DCL is structurally different from hotels: you book a *sailing* (a specific ship + departure date) and pick a *stateroom category* (a room type code like "O9C").

## The two DCL tools (in the order you call them)

| Step | Tool | Cost | Use it for |
|---|---|---|---|
| 1 | `cruise_search_sailings` | cheap (~20–60s) | Find sailings by month/ship. Returns sailing ID, ship, itinerary name, dates, nights, **ports of call**, lowest fare, available y/n. |
| 2 | `cruise_list_categories` | full quote (~15–30s) | After the user picks ONE sailing AND you've asked the placeholder question, render the FULL pricing view: cruise title, itinerary + ports + sea days, payment schedule (deposit-due rule per tier, final-payment date), sectioned tables (Interior / Oceanview / Oceanview with Verandah / Concierge) with deck/location/total/deposit/gratuities/grand-total, gratuities footer, and price-drop alert guidance. When `placeholder: true`, the tables show discounted Total / Savings / placeholder Deposit. **This is the only DCL pricing tool — there is no separate "quote" step.** |

**Cruise pricing has moved fully to the two tools above.** `explore_rates` is hotel-only — do not pass `sailingId` to it. The previous `list_cruise_sailings`, `list_stateroom_categories`, and `cruise_get_quote` tools have been removed (their functionality is now folded into `cruise_list_categories`).

## Three HARD rules the server enforces

### Rule A — Party MUST be confirmed BEFORE the very first cruise tool call

Before calling **any** cruise tool — `cruise_search_sailings` or `cruise_list_categories` — you MUST ask the user three things explicitly:

1. **How many adults?**
2. **How many children?** (age 0–17)
3. **The age of each child?** — one age per kid, in years. Kids under 3 get a discounted "infant" rate.

Ask these BEFORE `cruise_search_sailings`. Search results show per-category lowest pricing, which depends on the party composition; asking once at the start means you never have to re-ask at the categories step. The same party flows through to every subsequent cruise call.

Then pass `adults`, `children`, `childAges`, AND `partyConfirmed: true` on every cruise tool. The server REFUSES the call when `partyConfirmed` is missing or false; it returns a message listing the exact three questions you forgot.

**Never set `partyConfirmed: true` without having asked.** That defeats the whole guard — it exists specifically to prevent you from silently defaulting to "2 adults, 0 children, no childAges". The default is wrong half the time and the user is annoyed when you re-quote after they correct it.

### Rule C — Placeholder status MUST be confirmed before `cruise_list_categories`

Before calling `cruise_list_categories`, you MUST ask the user whether they're booking with a DCL placeholder (Booking a Future Cruise Onboard). The placeholder discount is 10% off the voyage fare and a $250-reduced deposit; standard pricing has neither. Disney never returns placeholder pricing in its APIs — the math happens in the tool, but only when you pass `placeholder: true`.

**Ask the placeholder question right after the user picks a sailing — BEFORE `cruise_list_categories`.** The category list itself renders prices, so showing the user standard pricing when they have a placeholder means they see the wrong numbers and you have to re-quote afterwards. Asking once upfront avoids that.

Phrase it like:
> *"Are you booking this with a Disney Cruise Line placeholder (also called Booking a Future Cruise Onboard)? It gets you 10% off the voyage fare and a $250 reduced deposit. Or is this your first DCL cruise / no placeholder?"*

Then pass BOTH:

- `placeholder: true` (yes, has placeholder) or `placeholder: false` (no placeholder) — required, no default.
- `placeholderConfirmed: true` — REQUIRED attestation flag.

The server REFUSES the call when `placeholderConfirmed` is missing or false; it returns a message naming exactly what to ask. **Never set `placeholderConfirmed: true` without having asked.**

**`cruise_list_categories` placeholder mode renders the same sectioned tables, plus a Savings column.** When `placeholder: true`:

- Same four sections (Interior / Oceanview / Oceanview with Verandah / Concierge), same per-row metadata (Decks, Location, Rooms).
- The `Total` column shows the **adjusted total** (10% off voyage fare).
- A new `Savings` column shows the per-category dollar discount vs Disney's standard pricing (positive dollar value, e.g. `$220.00`).
- The `Deposit` column shows the **placeholder deposit** (standard deposit minus $250 prepaid onboard).
- The Payment schedule says deposit is due **immediately** when the booking converts (no multi-day grace).

Render the table verbatim (Rule B applies). Don't summarize the savings; let the user see each row's discount.

### Rule B — List EVERY category, never "best value" or "top N"

When `cruise_list_categories` returns 27 categories, render all 27. The output is a complete markdown table — **paste it verbatim into your reply to the user**. The tool appends a `⚠ **RENDER THIS RESPONSE TO THE USER VERBATIM**` directive at the bottom of its text response; that directive is binding.

**Forbidden patterns** (Claude has been caught doing all of these — DO NOT):

- ❌ *"Here's the full category list for WD0105 — Disney Destiny… A few highlights:"* followed by 5 bullet points naming the cheapest in each tier. **No.** The full table goes in the reply, not "highlights".
- ❌ *"Cheapest inside: 11C at $4,472.64. Cheapest oceanview: 09D at $4,652.64. Cheapest verandah: 07A. Concierge entry: 03A."* This is the canonical anti-pattern — five categories listed when the table has 27. **Render the table.**
- ❌ "Top 5 cheapest…" → render every row.
- ❌ "Skipping Concierge since they're $$$" → Concierge categories sell out fast and the user wants to see them. They are NOT optional.
- ❌ Collapsing subcategories: "Inside Standard, Oceanview Standard, Verandah Standard, Concierge Suite" — every row in the table is a distinct stateroom code with its own price and room count. Flattening loses information.
- ❌ Replacing the table with prose ("Inside categories start at $X, Verandah at $Y…") → the user explicitly wants the grid.

**The correct pattern:**

1. Paste the **Itinerary** line, per-port bullets, and the **At Sea** bullet (when sea days > 0) at the top.
2. Paste the **Payment schedule** block — it shows the deposit-due rule per tier (standard vs concierge — they often differ; concierge typically due immediately, standard has a multi-day grace) and the final-payment due date that applies to all categories on the sailing.
3. Paste every section header and every markdown table verbatim. The four sections are **Interior**, **Oceanview**, **Oceanview with Verandah**, and **Concierge** — render them as `### Section` H3 headers with their tables underneath. Do NOT collapse to one combined table.
4. Each row carries: Category, Name, Decks, Location, Rooms, Total, Deposit, Gratuities (per category — Concierge tier uses a higher rate), Total + Gratuities. Render every column.
5. Render the gratuity-rate footer line and the price-drop alert threshold note too.
6. Add **exactly one** follow-up question after the tables: *"Want me to set a price-drop or availability alert on any of these categories?"*
7. Wait for the user to respond.

**Do NOT ask any of these** — `cruise_list_categories` already has the answer:

- ❌ *"Want me to pull a full quote with deposit due date, final payment due date, and ports of call?"* — all there.
- ❌ *"Which categories should I quote?"* — only ask this after the user has signaled they want the placeholder math; the default next step is the alert offer.
- ❌ *"Want full pricing?"* — the prices, deposit, gratuities, and grand total are in the rows.
- ❌ *"Should I check the placeholder discount?"* — only ask this if the user explicitly wants the placeholder-adjusted numbers AFTER they've picked categories. It is NOT the default follow-up.

The only time `cruise_list_categories` runs after `cruise_list_categories` is when the user explicitly wants the placeholder math (Booking a Future Cruise Onboard — 10% off + $250 reduced deposit). At that point, ask the placeholder yes/no question, then call `cruise_list_categories` with the user's picks. Otherwise the natural next step from the tables is the alert offer.

That's it. No prose summary. No highlights. No tier "entry points". The user is making a real comparison decision — even rows that look obviously too expensive or obviously too cheap inform the decision. Render every priced row.

**After the user picks a subset:** they might say "just give me the cheapest Verandah" or "skip Concierge". Then you re-render a filtered view from the data you already have — no new tool call needed. But the **FIRST** render is always the full list.

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

Pass the 2-letter code as the `ships` filter on `cruise_search_sailings` when the user wants a specific ship.

## Stateroom category shape (4 broad types)

| Type | Letter prefix | Examples | Typical vibe |
|---|---|---|---|
| Inside | `I`, `11` | `11C`, `11B`, `11A` | No window, cheapest |
| Oceanview | `O`, `8`, `9` | `O9C`, `8D`, `9B` | Porthole or window, no verandah |
| Verandah | `V`, `4`, `5`, `6`, `7` | `V7A`, `5C`, `4A` | Private balcony |
| Concierge | `T`, `V1`, `V2`, `V3` | `V3B`, `T`, `R` | Concierge lounge access, top decks, pricier |

The exact category codes come from `cruise_list_categories` — **never guess**; always call the tool to get the current list for a specific sailing.

## Required follow-up questions for DCL

DCL questionnaire is different from hotels. Ask BEFORE calling the relevant tool:

### For sailing search (`cruise_search_sailings`)
1. **Party (REQUIRED)** — adults, number of children, each child's age. See Rule A — server refuses the call without `partyConfirmed: true`. Ask this BEFORE picking month/ship/nights.
2. **Month(s)** — `YYYY-MM` format, at least one. Ask "what month(s) are you flexible on?".
3. **Ship preference** — STRONGLY RECOMMEND picking at least one. Full-month-all-ships searches take 1–2 minutes; narrowing to one or two ships brings it to 20–40 seconds. Present the list: Wish (WW), Fantasy (DF), Treasure (DT), Destiny (WD), Dream (DD), Magic (DM), Wonder (DW), Adventure (DA). If the user says "any ship", warn them the search will take 1–2 minutes and proceed.
4. *(Optional)* **Nights** — Disney's API uses BUCKETS, not exact night counts: **1-3, 4, 5-6, 7, 8-13, 14-100**. The MCP tool auto-normalizes:
   - Bare integer: `nights: ["5"]` → server maps to "5-6" bucket → returns 5 AND 6 night sailings.
   - Bucket: `nights: ["5-6"]` → server adds the suffix → same result.
   - Full filter value: `nights: ["5-6;filterType=night"]` → passes through.
   There is NO exact "5 nights only" filter on Disney's side. If the user said "5 nights", warn that 6-night sailings will also appear in results — let them filter visually.
5. *(Optional)* **Destination** — Disney's API accepts the plain code directly (e.g. `["BAHAMAS"]`, `["CARIBBEAN"]`, `["ALASKA"]`, `["EUROPE"]`, `["MEXICAN RIVIERA"]`).
6. *(Optional)* **Ships** — pass the 2-letter code directly (`["WW"]`, `["WD"]`, `["DF"]`). No suffix needed.

**Search performance note:** `cruise_search_sailings` scrapes DCL live. Scope drives latency:
- One specific ship + one month → ~20 seconds
- Two or three ships + one month → ~40–60 seconds
- All ships + one month → ~120 seconds (close to the MCP tool-call limit)
- Multiple months without ship filter → often times out; split into per-month calls if needed

**When the user asks for a specific target date (e.g. "closest to April 8"):** the tool returns every matching sailing for the month filters, which can be 30–50 rows for a popular ship. Don't dump all of them. Instead:

1. Pick the tightest relevant month filter — one month is almost always enough; add the previous or next month only if the target date is within a week of the month boundary.
2. After the call returns, sort results by `|departureDate - targetDate|` and show only the **3–5 closest** sailings — nearest match first, then nearest alternatives on either side.
3. Say something like: *"Closest sailings to April 8, 2027: [list]. Want the full month of options, or pick one of these to price?"*

### For listing categories (`cruise_list_categories`)
Required: `sailingId`, party (adults, children, childAges) + `partyConfirmed: true`, AND `placeholder: true | false` + `placeholderConfirmed: true`. Both Rule A and Rule C apply — ask both questions before this call. The category list itself shows pricing, so the placeholder answer must be in hand before the first render.

### For the full quote (`cruise_list_categories`)
1. **Sailing ID** — from search results.
2. **Party confirmed** — adults, children, childAges (one age per child) + `partyConfirmed: true`. Required — see Rule A above. The server refuses the call without this.
3. **Placeholder confirmed** — `placeholder: true | false` + `placeholderConfirmed: true`. Required — see Rule C above. The server refuses the call without this. Ask: *"Are you booking with a DCL placeholder (10% off + $250 reduced deposit), or first/no placeholder?"*
4. **Categories the user is interested in** — pass `categoryCodes: ["04A","08A"]` to limit the deposit fan-out. Strongly preferred — call `cruise_list_categories` FIRST, show the user every category, let them pick, then quote their picks. Only omit `categoryCodes` when the user explicitly said "show me everything". Phrase it like:
   > *"Are you booking this with a Disney Cruise Line **placeholder** (also called Booking a Future Cruise Onboard)? It gets you 10% off the voyage fare and a $250-reduced deposit since you prepaid that on your prior cruise. Or is this your first DCL cruise / no placeholder?"*

   - If yes → call `cruise_list_categories({ ..., placeholder: true })`. The placeholder flag triggers the legacy detailed format that shows the adjusted prices.
   - If no → call without `placeholder` (or `placeholder: false`) so the user sees Disney's standard pricing in the new compact format.
   - Do not skip this question — it changes the numbers materially.

   ### Wait for the answer before calling `cruise_list_categories`. Don't speculatively fetch.

   Claude's instinct is to be helpful by saying *"while you answer, I'll start fetching"* and immediately calling the tool. **Don't.** The placeholder flag is part of the tool input — calling without it means the prices are wrong for placeholder users, and you'll have to recall the tool (each call is 15–30s with `categoryCodes`, longer without). That's a worse experience than waiting 5 seconds for the user's reply.

   What IS safe to do in parallel with the placeholder question:
   - `cruise_search_sailings` (only if you don't already have the sailingId).
   - `cruise_list_categories` — listing categories doesn't take a `placeholder` param.
   - `get_my_profile` — for notification setup later.

   What is NOT safe to do until the user answers:
   - `cruise_list_categories({ sailingId, ... })` — placeholder is part of the input.
   - Any pricing-presentation work (no point until you have prices).

4. **Do NOT ask:** ticket days, dining plan, Memory Maker, travel protection — DCL doesn't offer these as toggles. The cruise fare already includes onboard dining (rotational dining + Cabanas + QSRs). Gratuities are extra but NOT configurable through the tool.

### Deposit-due timing differs by mode

- **Without placeholder:** Disney's standard window applies. The "Deposit Due" column says "At booking" or "Within X days of booking" based on what Disney returned.
- **With placeholder:** the placeholder deposit is due **immediately** on the day the booking is converted from the placeholder. There's no multi-day grace period — DCL requires payment same-day. The placeholder format labels the deposit row "due immediately"; surface that.

### Final payment date is REQUIRED in your output, both modes

The standard format puts the final-payment date in the **Final Payment Due** column of the table. The placeholder format emits a callout line above the per-group blocks like:
> 📅 **Final payment due Jan 5, 2027** for all priced categories on this sailing.

In either format, **you MUST keep the final-payment date visible in your reply**. It's the single most important piece of payment-timing info for the user. Do not drop, summarize, or paraphrase as "due before sailing" — the user wants the exact date.

### How placeholder pricing actually works

**Disney's APIs do NOT return placeholder pricing.** Disney has no idea which user holds a placeholder; they always return standard fare/tax/total. The placeholder math is computed inside the tool using Disney's exact pricing as the input. The formula is:

```
netFare        = total - tax
discount       = netFare × 0.10
adjustedTotal  = total - discount
adjustedFare   = adjustedTotal - tax
standardDep    = adjustedFare × 0.10
placeholderDep = max(0, standardDep - 250)
```

When you call `cruise_list_categories({ ..., placeholder: true })`, the tool's text output **already contains** the adjusted values. The summary table headers swap to `Total (10% off)` and `Placeholder Deposit`, and each detail block adds `10% off fare`, `Adjusted total`, `Adjusted fare`, `Standard deposit`, and `Placeholder deposit` rows. **Render those rows as-is.**

### NEVER write any of these:

- "The server returned null for placeholder-adjusted pricing"
- "The tool was unable to apply the 10% fare discount automatically"
- "Disney's API didn't return placeholder data"
- "Confirm the exact numbers with DCL or a travel agent before booking" (specific to placeholder — Disney won't quote placeholder prices)
- Any other note implying the placeholder math was attempted server-side and failed

These are wrong. Disney never returns placeholder pricing, by design. The math happens in the tool. If the placeholder rows aren't in your output, it's because you forgot to pass `placeholder: true` — re-call the tool with the flag set. Don't write an excuse.

## Workflow — "what Disney cruises are available in Sep 2026?"

```
0. ASK PARTY FIRST. "How many adults? How many children? Each child's age?"
   Wait for the answer before any tool call.

1. cruise_search_sailings({
     months: ["2026-09"],
     ships: ["WW"],            // narrow if user has a preference
     nights: ["5"],            // optional — server normalizes 5 → 5-6 bucket
     adults: 2, children: 0, childAges: [],
     partyConfirmed: true,     // REQUIRED — only after asking
   })

   --- After user picks a sailing, ASK THE PLACEHOLDER QUESTION before
       cruise_list_categories. The category list shows pricing; asking
       afterwards means the user saw the wrong numbers.
2. Render the sailing list verbatim (the tool returns a markdown table with
   sailing ID, ship, itinerary, dates, nights, PORTS OF CALL, available).
3. Ask: "Which sailing looks interesting?"
4. cruise_list_categories({
     sailingId: "WW0509",
     adults: 2, children: 0, childAges: [],
     partyConfirmed: true,           // REQUIRED — only after asking party
     placeholder: true | false,       // REQUIRED — pass user's actual answer
     placeholderConfirmed: true,      // REQUIRED — only after asking placeholder
   })
   - Renders itinerary line + per-port bullets at the top
   - Then the category list (no deposits, no due dates — fast view)
5. Ask: "Which categories should I quote? Pick 1–3 and I'll pull the full
   pricing with deposit and due dates." Also ask the placeholder question
   here if you haven't already.
6. cruise_list_categories({
     sailingId: "WW0509",
     adults: 2, children: 0, childAges: [],
     partyConfirmed: true,             // REQUIRED — only after asking party
     placeholder: true | false,         // REQUIRED — pass user's actual answer
     placeholderConfirmed: true,        // REQUIRED — only after asking placeholder
     categoryCodes: ["08A", "11C"],    // ALWAYS pass — set to user's picks
   })
7. Render the quote output verbatim.
```

### Why the cruise quote step is split from the category list

`cruise_list_categories` calls Disney's `stateroom-availability` endpoint and returns categories + base prices in one shot (~10s). `cruise_list_categories` does the same, then ALSO fans out to Disney's `payment-calculator` endpoint once per category to get the deposit and final-payment date (~1–2s per category). When the user only cares about 2 categories, `categoryCodes: ["08A","11C"]` saves you ~20+ seconds of unnecessary API calls.

When the user explicitly says "show me every category with full pricing", omit `categoryCodes` and accept the slower call.

## Workflow — "alert me if the Wish 4-nighter in Sep drops below $3200"

1. Find the sailing via `cruise_search_sailings`.
2. Find the category via `cruise_list_categories` → get `category`, `stateroomType`.
3. Recap: "Watching Disney Wish 4-Night Bahamian Sep 7–11, 2026, Category O9C (Deluxe Oceanview Stateroom), 2 adults. Price-drop alert fires when Disney's voyage fare drops under $3,200 (gratuities excluded — those are auto-charged at end of cruise and aren't part of the fare). Confirm?"
4. `create_alert({ alert: { kind: "cruise", sailingId, stateroomCategory: "O9C", stateroomType: "OUTSIDE-DELUXE", shipName: "Disney Wish", sailingName: "4-Night Bahamian Cruise", checkIn: "2026-09-07", checkOut: "2026-09-11", adults: 2, children: 0, childAges: [], alertType: "price_drop", currentPrice: 3200, notificationMethod: "email", clientNote: "" } })`

### Cruise price-drop alert thresholds — use the `Total` column, NEVER `Total + Gratuities`

The Room Genie worker watches Disney's voyage fare on every re-check (fare + tax, **gratuities excluded**). When you propose a price-drop alert, the threshold MUST come from the `Total` column of the `cruise_list_categories` output, not the `Total + Gratuities` column. Mistakes:

- ❌ "I see the grand total is $4,632.64 — I'll set the alert at $4,632.64." → that's `Total + Gratuities` ($4,472.64 fare + $160.00 gratuities). The worker compares against $4,472.64, which is already below $4,632.64 → alert fires on the next check. Useless alert.
- ✅ "I see the Total is $4,472.64 — I'll set the alert at $4,472.64 (today's exact voyage fare; gratuities are charged separately)." Worker compares Disney's voyage fare against $4,472.64; fires only if it actually drops.

When recapping the alert in chat, say something like *"price-drop alert fires when Disney's voyage fare drops under $X (gratuities excluded — those are auto-charged at end of cruise)"* so the user knows what's being watched and isn't surprised when their gratuities bill is on top.

## DCL-specific gotchas

- **Sailing IDs** look like `WW0509` (Wish, sailing 0509). Don't invent them — always pull from `cruise_search_sailings`.
- **Child ages affect pricing** — kids under 3 sail at a discounted rate; collect ages explicitly.
- **Concierge cabins are very limited** — `roomCount` is often 2–5 for Concierge categories. Sellout risk is real; an availability alert is valuable.
- **Port Canaveral embarkation** is the default for the Florida-based ships (Wish, Fantasy, Dream, Treasure, Destiny). European/Alaskan sailings use regional ports (Barcelona, Vancouver, etc.).
- **Special offers** (Kids Sail Free, military rates, Florida resident) show up in `cruise_search_sailings` response when applicable. Surface them when present.
- **Gratuities are not included** in the cruise fare. They're auto-charged at the end of the cruise at the rate Disney returns per-sailing (typically $14.50–$16.50 per guest per night). Don't promise "all-inclusive."

## Rendering `cruise_list_categories` output

**Render the tool's `content[0].text` verbatim.** Do NOT re-format, summarize, or drop sections. The new compact format produces this exact structure (when `placeholder` is false or omitted):

```
## {Cruise Title}
{Ship Name} · {departureDate} → {returnDate} · {numberOfNights} nights
Party: {N} adults{, M children (ages …)}

**Itinerary:** {nights} · {N} ports · {seaDays} sea days
- **{Port 1}** — {description}
- **{Port 2}** — {description}
- …

| Category | Type | Total | Deposit | Final Payment | Deposit Due | Final Payment Due |
|---|---|---|---|---|---|---|
| 04A | Concierge 1-Bedroom Suite | $9,432.18 | $1,500.00 | $7,932.18 | At booking | 2026-06-04 |
| 08A | Deluxe Verandah | $5,210.44 | $250.00 | $4,960.44 | At booking | 2026-06-04 |
| … |

**Gratuities** (auto-charged at end of cruise, not in totals above):
- Standard categories: $14.50/guest/night × 7 × 4 = $406.00
- Concierge categories: $16.50/guest/night × 7 × 4 = $462.00
```

Rules for the new format:

- **Keep every column.** Don't collapse to "Total + Deposit only" — the user explicitly wants Final Payment, Deposit Due, and Final Payment Due visible.
- **Keep the gratuities block.** It shows the per-day rate AND the trip total. Don't replace with "~$14.50/guest/night" — the trip total is what the user actually owes.
- **Keep the itinerary line + port bullets.** Ports of call are a major cruise decision input.
- **A "Quote limited to: …" footer** appears when `categoryCodes` was passed. That tells the user there are more categories on the sailing they could explore via `cruise_list_categories` — keep that footer.
- **A "sold out: {codes}" footer** appears when some categories on the sailing are sold out. Keep it; the user may want an availability alert on a sold-out cabin type.

### When `placeholder: true` is passed

The legacy detailed format is used (it has the placeholder math). Render it verbatim — the table has different headers (`Total (10% off)`, `Placeholder Deposit (due now)`), and there's a fenced detail block per category with `Adjusted total`, `Adjusted fare`, `Standard deposit`, `Placeholder deposit`, etc. The placeholder format also emits the `📅 **Final payment due …**` callout line above the blocks. Keep the callout. Keep every row.

### Every price in the output is EXACT. NEVER prefix with `~`.

- Totals, fares, taxes, deposits, final payments, gratuities — all come from Disney's APIs (availability + payment-calculator + sailing-details enrichment).
- `gratuitiesRate` and `conciergeGratuitiesRate` are Disney's actual per-sailing rates, not rounded averages.
- **Never write `~$X,XXX` or `approximately $X,XXX` or `about $X,XXX`** when rendering a number from the tool. They are exact figures for this sailing + party + category combination.
- The only exception: if Disney genuinely returned null for a field and the tool noted the fallback, surface that as `not returned by Disney` — never invent a tilde.

### NEVER recompute Disney's deposit, tax, gratuity, or final payment amounts

Those numbers come from Disney's own payment-calculator API via the tool and are already in the text output (and in `structuredContent.categories[].deposit` / `.price.tax` / the `gratuitiesRate` field). Disney's deposit, gratuity, and final-payment formulas vary per sailing, per category, and per concierge tier — there is no single percentage you can apply. Pass through whatever the tool emitted. Full stop.

### No Claude-synthesized preamble or postscript about gratuities

Do NOT write intro text like "Gratuities auto-apply at Disney's default rate: $80/guest standard, $136.25/guest concierge (sailing total, ~$16/night and ~$27.25/night)" before the table. The Gratuities block already carries that information with the EXACT rate and trip total for the sailing. Inventing a summary preamble introduces:

- Rounding/tildes that contradict the exact numbers below
- Rate figures that Claude estimates rather than reads

If the user asks specifically for the gratuity rate, cite the value from the tool's structuredContent (`gratuitiesRate` or `conciergeGratuitiesRate`) — never a computed average.

## Suggest next steps after retrieving DCL data

**After `cruise_search_sailings` returns multiple sailings:**
- "Want me to see the categories on any of these? Pick a sailing ID and I'll pull the cheap overview first — it's about 10 seconds."
- If many similar options: "The [ship] on [date] is usually the sweet spot for price — want me to dig into that one first?"
- If the user hasn't picked a ship yet: "Want to narrow to a specific ship? The Wish and Treasure are the newest and usually priciest."

**After `cruise_list_categories` returns categories for a sailing:**
- "Inside categories are cheapest — $X total here. Want a full quote (with deposit + final payment) on a specific category? I'll need to know about the placeholder discount first."
- If a Concierge category is still available: "Only [N] Concierge cabins left at this price — want a full quote on that one, or set an availability alert in case it sells out?"
- "Pick 1–3 categories and I'll pull the full quote — much faster than quoting all of them."

**After `cruise_list_categories`:**
- "Want a price-drop alert on a specific category at today's exact total ($X)? You can override the threshold if you want a cushion."
- "Want to check a nearby sail date? Shifting by a week or swapping ships often saves meaningfully."
- If the user only quoted 1–2 categories: "Want me to quote any other categories on this sailing? I have the sailing details cached."

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

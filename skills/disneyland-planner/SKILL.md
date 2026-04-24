---
name: disneyland-planner
description: Use this skill BEFORE calling `explore_rates` or `create_alert` for Disneyland Resort in California — any of the three on-property hotels (Disneyland Hotel, Grand Californian Hotel & Spa, Pixar Place Hotel / Paradise Pier, Villas at the Grand Californian) or any conversation about Disneyland Park, California Adventure, DCA, Downtown Disney, Anaheim. Contains the DLR follow-up questionnaire (ticketDays 1-5 NOT 2-10, narrower ticket set — no water-parks-sport or plus, NEVER ask about a dining plan since DLR doesn't have one, NEVER ask about Memory Maker since DLR doesn't support it, only Travel Protection), and cross-hotel comparison patterns. Do NOT use for Walt Disney World in Florida — use disney-world-planner for that instead.
---

# Disneyland Resort — trip planner

Layer this on top of `room-genie-core`. Disneyland Resort has fewer moving parts than WDW — only 3 hotels and different ticket structures.

## Hotels (all 3 are Disney-owned, on-property)

| Hotel | Vibe | Notable |
|---|---|---|
| **Disneyland Hotel** | Classic Disney, themed towers (Adventure, Fantasy, Frontier) | Closest monorail station, most iconic |
| **Grand Californian Hotel & Spa** | Arts-and-Crafts lodge, private DCA entrance | Premium tier, direct walk into California Adventure |
| **Pixar Place Hotel** | Pixar-themed (formerly Paradise Pier) | Walking distance to DCA via Grand Cal, lowest price of the three |

There is no "Value" or "Moderate" category at DLR — all three are premium. If a user wants budget, they'll need a Good Neighbor hotel (not in Room Genie).

## Ticket types

**Labels to show the user** (DLR has a slimmer set than WDW — only these two in the consumer pick list):

- **"1 Park Per Day"**
- **"Park Hopper"**

Never show the API enum (`no-option`, `park-hopper`) in chat. Never show "Base" — that's not a label we use. `genie-plus` and `park-hopper-genie-plus` exist in the API but are not offered in the consumer questionnaire; `water-parks-sport` and `plus` are WDW-only. Default to "1 Park Per Day" if the user hasn't expressed a preference.

## Dining

Disneyland **does not offer a dining plan** like WDW does. Always use `diningPlan: "none"` for DLR. If the user asks about a dining plan, clarify that DLR doesn't have one.

## Memory Maker

DLR does **not** offer Memory Maker as a package add-on through the Disney booking flow. Always pass `memoryMaker: false` for DLR. Do NOT ask the user about it — if they ask, explain Memory Maker is WDW-only in the package flow (DLR's equivalent PhotoPass product is purchased separately on-site or in the app).

## Travel Protection

Same as WDW — $99/adult, children free. Offer it for longer stays.

## Typical DLR trip shape

- 3-day ticket is the most common (covers both parks with a hopper).
- Stays are shorter than WDW — 2–4 nights is typical, often as part of a California trip.
- Peak season: spring break, mid-summer, Halloween Time (Sep–Oct), Holidays (mid-Nov to early Jan).

## Required follow-up questions for DLR (ask before calling `explore_rates`)

DLR has fewer knobs than WDW. Ask dates FIRST, then package specifics.

### Ask dates first

1. **Check-in and check-out dates** (YYYY-MM-DD).
2. **Party** — adults, number of children, each child's age.
3. **Package or room-only?** If unclear from the user's phrasing.

### If `package`, ask the rest — in this shape:

4. **Number of park days** (integer 1–5). DLR maxes at 5, UNLIKE WDW. 3 is the most common DLR length.

5. **Ticket type** — show the user EXACTLY these two labels, verbatim:
   - **"1 Park Per Day"**
   - **"Park Hopper"** — most DLR guests pick this since the parks are next to each other

   Never show "Base" and never show the API enum (`no-option`, `park-hopper`) in user-facing text. Park Hopper Plus and Waterpark & Sports are WDW-only; Lightning Lane variants are in the API but are NOT offered in the DLR consumer questionnaire — do not surface them unless the user explicitly asks.

6. **Dining plan:** DO NOT ASK. DLR has no dining plan — always pass `diningPlan: "none"` silently. If the user asks for one, explain DLR doesn't offer one.

7. **Memory Maker:** DO NOT ASK. DLR has no Memory Maker package add-on — always pass `memoryMaker: false` silently. If the user asks, explain it's WDW-only in the package flow.

8. **Travel Protection?** — ask as a **separate** yes/no question. $99/adult, children free. Default no. This is the ONLY paid add-on to offer at DLR.

### If dates are missing

Ask for dates + party first, then come back with the package questions.

## Workflow (same pattern as WDW, narrower inputs)

1. `list_resorts({ query: "Disneyland" })` — returns the 3 DLR hotels (plus Pixar Place if listed under its new name).
2. **`list_room_types({ resortId })` — REQUIRED before pricing.** Show the user the room list and ask which they want priced. DLR hotels have fewer rooms than WDW (usually 4–8), but each is still a separate cart flow — don't default to all-rooms. Let the user pick 1–3.
3. User picks rooms → record the `roomTypes` UUID array.
4. Collect DLR follow-up answers above (dates, party, package vs room-only, ticket days / type, Travel Protection — no dining plan, no Memory Maker).
5. `explore_rates({ mode: "package", roomTypes: [<picked ids>], ... diningPlan: "none", memoryMaker: false })`.
6. Render result with deposit + balance + due dates. Offer price-drop alert if interesting.

## DLR-specific gotchas

- **No dining plan** — reiterate if the user asks.
- **No Memory Maker** — DLR doesn't offer Memory Maker as a package add-on. If the user asks, explain it's WDW-only in the package flow; PhotoPass at DLR is bought separately on-site.
- **Pixar Place Hotel** was renamed from Paradise Pier in 2024; `list_resorts` may return either or both names depending on DB state. Use the returned name.
- **Grand Californian has DVC villas** (Villas at the Grand Californian) — a separate resort record. Disambiguate if the user wants the villas specifically.
- **Ticket days**: DLR tickets max out at 5 days. Reject or warn if the user asks for a 7-day DLR ticket.

## Suggest next steps after retrieving DLR data

**After a Grand Californian quote:**
- "Want to see Disneyland Hotel for the same dates? Usually $100–200/night cheaper with a similar experience."
- "Check Pixar Place — it's typically the lowest-priced of the three on-property hotels and still walkable to DCA."
- "Add the Park Hopper if you're not already? DLR is small enough that most guests hop between parks daily."

**After a Disneyland Hotel quote:**
- "Compare against Grand Californian — it's pricier but you get the private DCA entrance."
- "Or Pixar Place for a cheaper on-property option?"
- "Want to try a 3-day ticket vs your current days? 3-day is the most common DLR length."

**After a Pixar Place quote:**
- "Want to see what the step up to Disneyland Hotel looks like? Often $200+/night more but gets you the classic Disney experience."
- "Grand Californian is the premium option if the budget allows."

**After `explore_rates` shows everything sold out at DLR:**
- "Try ±1 week — DLR hotel inventory is tight, but a small date shift often helps."
- "Set an availability alert? DLR cancellations happen often, especially 30–60 days out."

**After a full package quote with tickets:**
- "Want to try without the Lightning Lane (`genie-plus`) add-on to see the baseline?"
- "Add a day — sometimes the per-day ticket rate drops meaningfully on longer tickets."
- "Set a price-drop alert at today's exact price? You can override if you want a buffer."

**After `list_room_types` at a DLR hotel:**
- Surface the Standard View vs Premium View (or themed rooms like Pirates of the Caribbean at Disneyland Hotel) and explain the tradeoff in one line each.
- Avoid recommending Concierge unless the user's budget signals it.

**When the user seems torn between WDW and DLR:**
- Offer: "Want me to quote the same party at a comparable WDW resort so you can compare totals?" (e.g., Grand Californian ↔ Grand Floridian)

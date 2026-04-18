---
name: disneyland-planner
description: Disneyland Resort (California) trip planning — picking a hotel, quoting packages, and setting alerts. Use when the user mentions Disneyland Resort, DLR, Disneyland, California Adventure, DCA, Downtown Disney, Anaheim, or the specific DLR hotels — Disneyland Hotel, Grand Californian, Pixar Place Hotel (formerly Paradise Pier). Use when the user is clearly talking about the California parks, NOT Walt Disney World in Florida.
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

## Ticket types (DLR uses a slimmer set than WDW)

Pass these values as `ticketType`:

- `no-option` — one park per day (either Disneyland or California Adventure, chosen day-of)
- `park-hopper` — hop between Disneyland and DCA same day
- `genie-plus` — Lightning Lane (DLR equivalent of Genie+)
- `park-hopper-genie-plus` — hop + Lightning Lane

**Don't use**: `water-parks-sport` or `plus` — those are WDW-only.

## Dining

Disneyland **does not offer a dining plan** like WDW does. Always use `diningPlan: "none"` for DLR. If the user asks about a dining plan, clarify that DLR doesn't have one.

## Memory Maker

- **PhotoPass+ / Memory Maker** is available at DLR but priced differently from WDW. Our `explore_rates` handles DLR's pricing automatically — set `memoryMaker: true` if the user wants it.

## Travel Protection

Same as WDW — $99/adult, children free. Offer it for longer stays.

## Typical DLR trip shape

- 3-day ticket is the most common (covers both parks with a hopper).
- Stays are shorter than WDW — 2–4 nights is typical, often as part of a California trip.
- Peak season: spring break, mid-summer, Halloween Time (Sep–Oct), Holidays (mid-Nov to early Jan).

## Workflow (same pattern as WDW, narrower inputs)

1. `list_resorts({ query: "Disneyland" })` — returns the 3 DLR hotels (plus Pixar Place if listed under its new name).
2. `list_room_types({ resortId })` — room categories vary (standard view, premium view, concierge, suites, themed rooms).
3. Collect dates + party + ticketDays (3 is common) + ticketType.
4. `explore_rates({ mode: "package", ... diningPlan: "none" })`.
5. Render result. Offer price-drop alert if interesting.

## DLR-specific gotchas

- **No dining plan** — reiterate if the user asks.
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
- "Set a price-drop alert under ~$X?" (5–10% below current)

**After `list_room_types` at a DLR hotel:**
- Surface the Standard View vs Premium View (or themed rooms like Pirates of the Caribbean at Disneyland Hotel) and explain the tradeoff in one line each.
- Avoid recommending Concierge unless the user's budget signals it.

**When the user seems torn between WDW and DLR:**
- Offer: "Want me to quote the same party at a comparable WDW resort so you can compare totals?" (e.g., Grand Californian ↔ Grand Floridian)

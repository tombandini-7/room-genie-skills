---
name: aulani-planner
description: Use this skill BEFORE calling `explore_rates` or `create_alert` for Aulani, A Disney Resort & Spa in Ko Olina, Hawaii (Oahu), or any conversation about a Disney Hawaii trip. Aulani has NO theme park tickets, NO dining plan, NO Memory Maker — its questionnaire is RADICALLY shorter than WDW/DLR (just dates + party + Travel Protection yes/no). Always use `mode: "availability"` or `mode: "room-only"`, never `mode: "package"`. Contains the Aulani-specific questionnaire, view category tradeoffs (Island View / Partial Ocean / Ocean View), and hotel-room vs villa math. Prevents Claude from asking about tickets or dining plans that Aulani doesn't sell.
---

# Aulani — trip planner

Layer this on top of `room-genie-core`. Aulani is Disney's Hawaii resort — simpler than WDW/DLR because it doesn't sell theme park tickets or dining plans as add-ons.

## Room categories at Aulani

- **Standard Hotel Rooms** — Island View, Partial Ocean View, Ocean View, Poolside
- **Deluxe Studios** (DVC) — Island View, Partial Ocean View, Ocean View
- **1-Bedroom Villas** — full kitchen, washer/dryer; various views
- **2-Bedroom Villas** — sleeps 8–9
- **3-Bedroom Grand Villas** — largest, priciest (2 at the resort)

Use `list_room_types({ resortId })` to get the exact IDs and names — don't hardcode.

## What's DIFFERENT about Aulani

Aulani has **no ticket, dining plan, or Memory Maker add-ons** through the Disney package flow:

- **`ticketDays` must be 0** — there are no theme park tickets to bundle.
- **`diningPlan` must be `"none"`** — no dining plans.
- **`memoryMaker` must be `false`** — no photo pass add-on in the package flow.
- **`travelProtection`** — this is the ONE add-on Aulani supports. $99/adult, children free. Offer it.

If a user asks for "a package with tickets and dining" at Aulani, correct them: Aulani doesn't offer those. Rates are room-only (or room + travel protection).

## Explore-rates modes at Aulani

- `mode: "availability"` — "is this room type open these dates?" — lightweight, no subscription needed.
- `mode: "room-only"` — full per-room pricing. Works on Aulani, uses the Aulani-specific backend automatically. Requires Explorer subscription.
- `mode: "package"` — NOT applicable in the WDW sense. Treat any package-style request as `room-only` plus optional travel protection.

## Required follow-up questions for Aulani (ask before calling `explore_rates`)

Aulani has ONLY ONE paid add-on — don't ask about tickets, dining, or Memory Maker. Ask dates first.

### Ask dates first

1. **Check-in and check-out dates** (YYYY-MM-DD).
2. **Party** — adults, number of children, each child's age.

### Then ask the one paid add-on

3. **Travel Protection?** — ask as a **separate** yes/no question. $99/adult, children free. Aulani's only paid add-on in the package flow. Default no, but worth offering for longer or costlier stays.

### If dates are missing

Ask for dates + party first. Travel Protection is a short afterthought once dates land.

**Do NOT ask:**
- "Package or room-only?" — Aulani has no package option. Always use `mode: "availability"` or `mode: "room-only"`.
- Park days / ticket type — Aulani has no tickets.
- Dining plan — Aulani has no dining plan.
- Memory Maker — Aulani has no Memory Maker add-on.

If the user tries to specify any of those, gently correct them: *"Aulani doesn't bundle tickets/dining/Memory Maker — those are only offered for Walt Disney World and Disneyland Resort trips."*

## Workflow

1. `list_resorts({ query: "Aulani" })` → get Aulani's resortId (there's only one).
2. `list_room_types({ resortId })` → show category + view options.
3. Collect Aulani follow-up answers above.
4. `explore_rates({ mode: "availability" | "room-only", resortId, roomTypes: [...], checkIn, checkOut, adults, children, childAges, travelProtection })`.
5. Render the table with deposit + balance when present. Aulani rates are often in the $400–1200/night range depending on view and season.

## Aulani-specific gotchas

- **DVC points vs cash** — Room Genie tracks cash rates only. If the user has DVC points, redirect them to their DVC account; we can't help with points bookings.
- **Concierge / Club Level** — some villas have Club Level access; shows as a separate room type. Ask the user if they want Club Level before creating an alert.
- **Seasons vary wildly** — Christmas / New Year's weeks are 2–3× Aulani's low-season rate. Mention this if the user seems surprised by the price.
- **3 travelers and over in a Standard Room** — occupancy caps are strict. If the party has 4+ adults, push to a 1-bedroom villa.

## Suggest next steps after retrieving Aulani data

**After a Standard Hotel Room quote:**
- "Want to see what the step up to a Deluxe Studio (DVC) looks like? Often competitive on rate and you get a kitchenette."
- "Try the view category next to yours — Partial Ocean View is usually $30–80/night above Island View but the photos are very different."
- If the stay is 5+ nights: "A 1-Bedroom Villa might work out cheaper per-night-per-person for your party. Want me to quote one?"

**After a Villa (DVC) quote:**
- "Compare against a Standard Hotel Room or a Deluxe Studio — for shorter stays, the extra space may not be worth the jump."
- "Try a different view (Island → Partial Ocean → Ocean) — the ocean views are often worth the upgrade at Aulani specifically."

**After `explore_rates` shows everything sold out:**
- "Aulani has very limited inventory — try ±7 days, or set an availability alert and we'll ping you if cancellations open up."
- "Christmas, Thanksgiving, and Spring Break are the tightest weeks. Are you flexible on dates?"

**After a quote that made the user wince at the price:**
- "Want me to check early December or late January — Aulani's lowest-demand weeks outside hurricane season?"
- "Try dropping Travel Protection to see the base rate?" (only if they had TP on)
- "If the dates are flexible, I can check a 3- vs 5- vs 7-night stay — sometimes the per-night rate changes."

**After `list_room_types` at Aulani:**
- Surface 2–3 options spanning the price range: "Cheapest: Standard Hotel Room, Island View. Mid: Deluxe Studio, Partial Ocean. Premium: 1-Bedroom Villa, Ocean View." Ask which vibe matches.

**When the user mentions they're doing an island-hopping trip:**
- "Want to alert only on specific date windows when you'd be on Oahu? I can set multiple alerts for different week combinations."

**After `create_alert` at Aulani:**
- "Travel Protection is the only Aulani add-on — want it on the alert too, or leave it off for now?"
- "Want a second alert for a different view or category as a backup?"

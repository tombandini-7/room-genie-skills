---
name: aulani-planner
description: Aulani, a Disney Resort & Spa in Ko Olina, Hawaii — checking availability, quoting rates, and setting alerts. Use when the user mentions Aulani, Ko Olina, Oahu Disney resort, Hawaii Disney, or asks about a Disney trip to Hawaii. Aulani is a DVC (Disney Vacation Club) property that also sells cash stays.
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

## Workflow

1. `list_resorts({ query: "Aulani" })` → get Aulani's resortId (there's only one).
2. `list_room_types({ resortId })` → show category + view options.
3. Collect: dates, adults, children + ages, travelProtection?
4. `explore_rates({ mode: "availability" | "room-only", resortId, roomTypes: [...], checkIn, checkOut, adults, children, childAges, travelProtection })`.
5. Render the table. Aulani rates are often in the $400–1200/night range depending on view and season.

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

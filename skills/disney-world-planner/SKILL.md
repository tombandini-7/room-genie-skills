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

**Always ask:**
1. **Check-in and check-out dates** (YYYY-MM-DD).
2. **Party** — adults, number of children, and each child's age.
3. **Package or room-only?** If the user already said one, skip.

**If `package`, also ask:**
4. **Number of park days** (integer 2–10). WDW tickets start at 2 days.
5. **Ticket type** — offer the choice with a one-line description:
   - Standard (`no-option`) — one park per day
   - Park Hopper (`park-hopper`) — hop between parks same day
   - Park Hopper + Water Parks (`plus`)
   - Lightning Lane (`genie-plus`) — includes LL passes
   - Park Hopper + Lightning Lane (`park-hopper-genie-plus`)
6. **Dining plan** — offer: None, Quick Service (`quick_service`), Standard (`standard` — classic 1 table-service/night), Deluxe (`deluxe`). Default to None if the user shrugs.
7. **Memory Maker?** ($185 flat) — yes/no. Default no.
8. **Travel Protection?** ($99/adult, children free) — yes/no. Default no, but offer for trips >$3k.

Once you have all the answers, walk through the full-package workflow below.

## Full-package workflow

When the user says "quote me a full package at [resort]":

1. `list_resorts({ query })` → confirm resortId (disambiguate if multiple matches — Polynesian Resort vs Polynesian Villas).
2. `list_room_types({ resortId })` → show the user the choices, ask which view/bedding they want (or track all).
3. Collect the WDW follow-up answers above.
4. `explore_rates({ mode: "package", ... })` — this can take 2–4 minutes; warn the user.
5. Render the resulting table. Highlight: nightly rate, package grand total, deposit ($200 + travel protection), balance due, deposit due date, balance due date.
6. Offer to `create_alert` with `price_drop` if the user wants to be notified when the package drops below that price.

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
- "Want to set a price-drop alert targeting ~$X?" (5–10% under current)

**After `list_room_types` returns many options:**
- Pick 2–3 likely matches (e.g., "Standard View" vs "Theme Park View" vs "Club Level") and describe the price/experience tradeoff in one line each. Don't dump all 15 rooms.

**After `create_alert` for WDW:**
- "Want to quickly check what's available at that resort right now so we know the baseline?"
- "Interested in setting a second alert at a cheaper Moderate as a backup?"

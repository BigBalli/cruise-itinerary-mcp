---
name: cruise-research
description: Research cruises with the Cruise Itinerary tools - find sailings, read day-by-day itineraries, track weekly fare changes, size port traffic by month and read the Cruise Price Index - and report the results with the data's caveats (weekly fares are not live, a missing fare means not offered or sold out, berths are capacity not passengers, port capacity is monthly only, estimated days can be off by one). Use whenever the user asks about cruise sailings, ships, ports of call, cruise fares or prices, itineraries, or cruise ship traffic at a port.
---

# Cruise research with Cruise Itinerary

The `cruise-itinerary` MCP server gives read-only access to a weekly register of cruise lines, ships, ports and upcoming sailings, kept by cruise-itinerary.com. This skill is about using it well and reporting what it says honestly.

## Work in this order

1. **Resolve names first.** Call `resolve_entity` with the name the user gave ("Port Canaveral", "Symphony of the Seas", "NCL"). Every other tool takes the returned `id` (a slug), never a free-text name. Pass `type` when you know it (`port`, `ship`, `line`).
   - Confidence 1.0 is an exact name. Below about 0.6 is a guess: show the candidates and ask the user which one they mean instead of picking one.
   - A port can have near-duplicates (for example `skagway` and `skagway-alaska-usa`). Prefer the exact match, and say so if results look split.
2. **Find sailings** with `search_sailings`: filter by `ship`, `line`, `port` (calls there), `embark_port`, `region`, departure `from`/`to`, `min_nights`/`max_nights`, `cabin`, `max_fare`. Use `sort: "fare"` for the cheapest first. Page with `page` and `per_page` (up to 50); `meta.total` says how many match.
3. **Open one sailing** with `get_itinerary` and its `sailing_id` for the dated day-by-day plan, or with `cruise_id` for the route shared by all sailings of that cruise.
4. **Fare history** for one sailing: `get_price_history` with the `sailing_id`.
5. **Port traffic**: `port_month_capacity` with a port `id`, a `from_month` (YYYY-MM) and `months` (up to 24).
6. **Market-wide price movement**: `cruise_price_index`. `view: "latest"` gives the last 12 weeks for all lines and the largest lines; `view: "series"` gives one line, cabin and segment over time.

## Report the caveats every time they apply

- **Fares are weekly snapshots, not live prices.** They are captured every Monday; `meta.as_of` is the date of the latest load. Say "as of <date>" and never present a fare as bookable or current. Send people to the cruise line or a travel agent to book.
- **A missing fare (null) means the cabin was not offered or was sold out** at capture time. The data does not say which. Do not call it "sold out" or "unavailable" on its own.
- **Fares are the lowest fare per cabin class, in whole US dollars**, as the supplier listed it. Do not add taxes or convert currencies unless asked, and say so if you do.
- **Itinerary days have a confidence.** `day_confidence: "exact"` comes from the cruise line's published itinerary (embark and disembark days are always exact). `"estimated"` days were placed from the distance between ports and can be off by a day. When you give a date for an estimated call, say it is estimated. On a cruise, `itinerary_source: "official"` means day numbers come from the line; `"supplier"` means only the order of stops is known.
- **Berths are capacity, not passengers.** `port_month_capacity` sums each ship's lower-berth capacity over every ship-day in port. Real passenger numbers differ (ships sail above or below lower-berth capacity). Write "berths" or "passenger capacity", never "passengers" or "visitors".
- **Port capacity is monthly only.** There are no per-day counts, by design: many call days are estimated. Do not divide a month into days or present a "busiest day" from this data. `ship_days` counts a ship in port two days twice; `distinct_ships` counts each ship once.
- **Coverage.** Strongest for North American cruise lines. TUI Cruises/Marella, AIDA, Saga, Fred. Olsen and Hapag-Lloyd are not covered. If a user asks about those, say the data does not include them rather than reporting "no sailings".
- **The Price Index** is a chained same-sailing index of weekly fares, 100 on 2026-09-06. A series needs at least 30 matched sailings. It describes how fares moved, not how expensive a cruise is.

## Plans and errors

Each tool needs a minimum plan on the user's Cruise Itinerary account:

| Tool | Plan |
|---|---|
| `resolve_entity`, `cruise_price_index` (latest) | Free |
| `search_sailings`, `get_itinerary` | Developer |
| `get_price_history`, `port_month_capacity`, `cruise_price_index` (series) | Business |

When a tool answers with an error saying the plan is too low, tell the user plainly which plan the question needs and give the link in the message (https://cruise-itinerary.com/pricing). Do not retry the same call, and do not try to work around it with other tools. Each successful tool call counts as one API call toward the account's monthly allowance, so avoid calls you do not need: resolve once, reuse the ids, and ask for a sensible page size.

Errors about arguments (an unknown id, a bad date) are safe to fix and retry once.

## Good answers look like this

Placeholders in angle brackets stand for values from the tool results.

- "As of the <as_of> load, the lowest listed balcony fare on <ship> from <port> in <month> is $<fare>. These are weekly snapshots, not live prices."
- "<port> sees <berths> berths of ship capacity in <month> across <ship_days> ship-days from <distinct_ships> ships. That is capacity, not a passenger count."
- "The call at <port> on <date> is estimated from the route and could be a day earlier or later; the embark and return days are exact."

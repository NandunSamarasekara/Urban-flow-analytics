# Walkthrough

This is the story of how the project came together — in order, including the parts that didn't work the first time.

## 1. Framing the problem

Rather than treating this as "load a CSV, make some charts," the goal was to build something that looked like a real production analytics stack: a governed lakehouse with a medallion architecture (Bronze/Silver/Gold), and a BI layer connected live to it — not a one-off export.

Three business questions were chosen up front, because a datathon dashboard without a driving question is just a pile of charts:

1. **Where is traffic worst, and is it predictable or chaotic?** (Congestion & Bottleneck)
2. **Which neighborhoods are underserved by transit hubs?** (First-/Last-Mile Integration)
3. **How ready is the fleet for an EV transition?** (Decarbonization & EV Readiness)

**Stack chosen:** Databricks (Free Edition) + PySpark for the pipeline, Uber H3 for geospatial joins, Power BI for the dashboard.

## 2. Bronze: getting 5GB of CSVs into Databricks without the upload bottleneck

The 12 monthly CSVs lived in a shared Google Drive folder. Rather than downloading them locally and re-uploading through Databricks' UI (slow, and pointless for 5GB), `gdown.download_folder` pulled them **directly into a Unity Catalog volume** (`workspace.bronze.raw_files`). Bronze schema, Silver schema, and Gold schema were set up as separate Unity Catalog schemas under one `workspace` catalog from the start.

## 3. Silver: cleaning, and the constraint nobody mentioned

Building `trips_clean` meant defining `is_valid_trip`: trip duration between 1–240 minutes, distance between 0.1–100 miles, average speed ≤100 mph, and non-null pickup/dropoff coordinates.

Then came a real constraint: **the dataset has no vehicle identifier** — only `provider_code` (the dispatch company). The EV-readiness and fleet-utilization metrics needed a vehicle-day grain, so this had to be solved with a synthetic `vehicle_id` (details in [TECHNIQUES.md](TECHNIQUES.md)).

Two sub-problems were built into `03_reference_data.ipynb`:
- **Part A — hub matching:** for every trip, find the nearest transit hub to pickup and dropoff, using H3 geohashing.
- **Part B — vehicle assignment:** derive a synthetic vehicle count/ID per provider, filtered to only `is_valid_trip` rows (an early version didn't filter first, and it showed).

The result, `trips_enriched`, combines both and gets persisted as a Delta table.

## 4. The bug: dates from 2008

While validating the Gold layer, `dim_date` came out to **6,301 rows** — obviously wrong for a 365-day dataset. Digging in: `pickup_date` in the supposedly-clean data ranged from **2008-12-31 to 2026-04-01**. The `is_valid_trip` filter checked duration, distance, and speed — but never checked whether the *date itself* made sense.

Two options: patch the symptom downstream (filter dates in the Gold layer query), or fix it at the root. **Root-cause fix was chosen** — a date-range plausibility check (2025-04-01 to 2026-03-31) was added directly into `is_valid_trip`'s definition, in the same cell where the other validity rules already lived (Cell 19 of `02_silver_layer.ipynb`).

That meant a **cascade rebuild**: `02_silver_layer` → `03_reference_data` (hub matching + vehicle assignment, both depend on `trips_clean`) → `04_gold_layer` (all dims and facts). Not a quick patch, but the only way to be sure every downstream table agreed with the fix. Afterward, `dim_date` correctly showed 365 rows, and `fact_vehicle_daily` dropped by 4 rows (outlier vehicle-days that only existed because of the bad dates).

## 5. Gold: designing the business logic, not just the schema

The Gold layer is where the three business questions turned into actual metrics:

- **Congestion Slowdown %:** for each origin-destination pair, compare peak-hour average speed against that same pair's off-peak/overnight baseline. Peak/off-peak wasn't hand-picked by clock hours — it came from a **data-driven percentile split** of trip volume by hour, applied city-wide (one set of peak hours for every O-D pair, not customized per pair or borough).
- **EV Readiness:** a 300km daily range assumption, checked at **vehicle-day grain** (one row per vehicle per day, from `fact_vehicle_daily`).
- **Hub Connectivity:** distance from each zone's centroid to its nearest transit hub, via H3 k-ring search.

Two data-quality judgment calls got made — and documented in-notebook rather than silently resolved:
- Zone 57 (Corona)'s missing centroid: left as a documented gap.
- The 59 zones with no hub within the search radius: **kept as nulls**, treated as a real "no hub nearby" signal rather than forced to a value by widening the search ring. (This decision has a subtlety — see [TECHNIQUES.md](TECHNIQUES.md) on `is_hub_connected`.)

Final Gold layer: 5 dimensions (`dim_zone`, `dim_provider`, `dim_time_bucket`, `dim_transit_hub`, `dim_date`) and 3 facts (`fact_od_congestion`, `fact_vehicle_daily`, `fact_zone_hub_connectivity`), all validated for row counts and nulls.

## 6. Power BI: learning it from zero, on a live connection

This was the first time working with Power BI or any BI tool at all — so the dashboard build started from installing the software.

**Phase 1 — CSV import (learning the basics).** All 8 Gold tables were exported as CSVs and imported, just to get comfortable with the tool and verify every row count matched the Gold layer.

**Phase 2 — data modeling.** Built the relationships: `dim_zone` → `fact_od_congestion.origin_loc_id`, and a duplicated `dim_zone_destination` (the role-playing dimension pattern) → `fact_od_congestion.dest_loc_id`, plus the rest of the star schema. `dim_time_bucket` and `dim_transit_hub` were left standalone by design — they weren't wired into any fact table yet.

**Phase 3 — going "industrial."** CSV import was replaced with a **live Databricks SQL Warehouse connector** (Import mode) — closer to how a real BI deployment would connect, and it removes the CSV re-export step entirely when the Gold layer changes.

**Building the five pages**, roughly in this order:

1. **Congestion & Bottleneck** — KPI cards, a Top-10-worst-routes table, a borough bar chart, a speed scatter plot. The table needed a synthetic `Route` key (origin+destination concatenated) before Power BI's Top-N filter would rank *routes* correctly — filtering by a single zone name was silently ranking by *origin zone*, aggregating across all its destinations. Later, a `Trip Count` slicer was added, which surfaced a real insight: **Manhattan and EWR's high average slowdown was mostly driven by low-volume outlier routes** — filter to routes with 200+ trips, and **Bronx** becomes the real #1 for sustained congestion.
2. **Transit Integration** — connectivity and distance KPIs, plus a coverage-gap chart (`Zones Without Nearby Hub, by Borough`) built directly from the null-count finding in the Gold layer. The zone-hub distance map hit a wall: the organization's email couldn't authenticate against Azure Maps. Fell back to the classic (soon-to-be-retired) Map visual, and added a DAX-bucketed `Distance Band` categorical column to fake the color gradient that Azure Maps would have given for free.
3. **EV Readiness** — daily distance and range-compliance KPIs, a monthly trend line, and an EV-readiness-by-provider comparison. A copy-pasted KPI card left a stale field reference pointing at the wrong table; caught by comparing the visible number against its own label.
4. **Temporal Patterns** — weekday/weekend and day-of-week/month breakdowns. `Sort by Column` — the normal way to fix "Monday, Tuesday..." sorting alphabetically instead of chronologically — was blocked by a DirectQuery capacity/license restriction. Worked around it with zero-padded numeric-prefix label columns (`"01 - April"`, `"1 - Monday"`), which sort correctly as plain text with no special sort configuration needed at all.
5. **Executive Overview** — the closing page. Every KPI from the other four pages, plus a combo chart layering `Avg Slowdown %` (bars) against `Avg Distance to Hub` (line) by borough — the one chart that ties all three business questions into a single visual argument.

## 7. What this build taught

- A validity filter is only as good as the checks you remembered to put in it — `is_valid_trip` looked complete until a completely different kind of bad data (implausible dates) slipped straight through it.
- "Fix it at the root and rebuild the cascade" costs more time up front than patching downstream, but it's the only way every table tells the same story.
- Nulls aren't always missing data to fill in — sometimes (like the hub-connectivity gaps) they're the finding.
- A slicer isn't just a filter — the Trip Count slicer on the Congestion page turned into the single most interesting insight in the whole dashboard, almost by accident.


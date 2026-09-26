# Techniques

A topic-by-topic explanation of the engineering decisions behind the pipeline and dashboard — the *why*, not just the *what*. For the chronological build story, see [WALKTHROUGH.md](WALKTHROUGH.md).

---

## 1. Medallion Architecture (Bronze → Silver → Gold)

The pipeline uses Databricks' standard medallion pattern, implemented as three separate Unity Catalog schemas (`workspace.bronze`, `workspace.silver`, `workspace.gold`) rather than three folders of loosely-related tables:

- **Bronze:** raw CSVs, untouched, landed in a managed volume (`raw_files`). No transformation — this layer exists purely so the original source is always re-derivable.
- **Silver:** cleaned and enriched at trip grain (`trips_clean`, `trips_enriched`). This is where validity rules, hub-matching, and vehicle assignment live.
- **Gold:** business-ready, dimensionally modeled tables (star schema) — what the BI layer actually queries.

The point of the separation isn't ceremony: when the date-validity bug was found (see below), it was fixable in exactly one place (a Silver-layer filter definition) with a clear, mechanical path to propagate the fix forward — because nothing downstream had merged raw and clean logic together.

## 2. ELT, not ETL

Data was **loaded first, transformed in place** — raw CSVs went straight into a Databricks volume with no pre-processing, and every transformation from that point on is PySpark/Spark SQL running inside Databricks against Delta tables. This is the ELT pattern rather than classic ETL (transform-before-load), and it's a natural fit for a lakehouse: Delta Lake gives ACID transactions and schema evolution, so each layer can be independently re-run and validated without re-touching the source files.

## 3. H3 Geohashing for Hub-Proximity Matching

**The problem:** for every trip, find the nearest transit hub — evaluated against 445 hubs, for every one of ~1.7M+ trip locations. A brute-force haversine distance calculation (every point against every hub) doesn't scale cleanly in Spark.

**The solution:** Uber's **H3 hierarchical hexagonal geospatial index**. Every point (trip pickup/dropoff, and every hub) is encoded to an H3 cell at **resolution 8** (~0.46 km² hexagons, roughly a 1km search radius). Instead of computing distance to all 445 hubs, the search uses `h3_kring(h3_col, 1)` — the target cell plus its immediate ring of neighbors — and only evaluates hubs whose H3 cell falls in that neighborhood. This turns an O(n × m) distance problem into an indexed lookup, which is what makes it tractable at Spark scale.

**Resolution 8 was a deliberate trade-off**, not a default: fine enough to distinguish "a hub is a short walk away" from "a hub is a subway ride away," coarse enough to keep the k-ring search radius (~1km) meaningful for what "connected" means in this dataset.

## 4. Data Quality Engineering

Two practices show up repeatedly in this pipeline:

**Defensive multi-condition validity gates.** `is_valid_trip` isn't a single check — it's a conjunction: duration bounds, distance bounds, speed ceiling, non-null coordinates, and (after the bug fix) a date-plausibility check. Each condition catches a different failure mode; none of them alone would have caught the corrupt-date problem, which is exactly why it slipped through until it was checked for explicitly.

**Root-cause fixes over downstream patches, with cascade rebuilds.** When the date bug surfaced in `dim_date`'s row count, the fix went into the *original* filter definition (`is_valid_trip` in `02_silver_layer.ipynb`), not into a `WHERE` clause bolted onto the Gold layer query. The cost of that choice is a full rebuild of everything downstream (`03_reference_data.ipynb` → `04_gold_layer.ipynb`) — but the payoff is that every table in the lakehouse agrees on what a valid trip is, instead of each layer carrying its own patched-on exception.

## 5. Dimensional Modeling (Star Schema)

The Gold layer is a conventional star schema: **5 dimensions**, **3 facts**, each fact table with a clearly defined grain:

| Fact table | Grain | Key business logic |
|---|---|---|
| `fact_od_congestion` | one row per origin–destination pair | peak vs. baseline speed comparison |
| `fact_vehicle_daily` | one row per vehicle per day | EV range compliance, daily distance |
| `fact_zone_hub_connectivity` | one row per zone | nearest-hub distance, connectivity flag |

**Role-playing dimension pattern.** `dim_zone` needs to join to `fact_od_congestion` twice — once as the *origin* zone, once as the *destination* zone. Power BI (like most BI tools) can't have two active relationships from one dimension table to one fact table without ambiguity, so the standard fix is to physically duplicate the dimension: `dim_zone_destination` is a copy of `dim_zone`, joined to `fact_od_congestion.dest_loc_id`, while the original `dim_zone` stays joined to `origin_loc_id`. Same data, two roles.

## 6. Metric Design Decisions

Three metrics needed a definition invented specifically for this project, not looked up from a standard formula:

**Congestion Slowdown %** — for a given O-D pair, `(baseline_avg_speed − peak_avg_speed) / baseline_avg_speed`. "Peak" and "baseline" (off-peak/overnight) hours weren't hand-picked — they came from a **data-driven percentile split** of trip-volume-by-hour across the whole dataset, applied uniformly city-wide. This was a deliberate scope decision: a per-borough or per-pair peak-hour definition would arguably be more precise, but it would also make every route's slowdown % incomparable to every other route's — the metric only works as a league table because "peak" means the same thing everywhere.

**EV Range Readiness** — a flat 300km daily range assumption, evaluated at vehicle-day grain (`is_under_ev_range` in `fact_vehicle_daily`). Vehicle-day, not trip-level, because range anxiety is a whole-day planning problem, not a single-trip one.

**Hub Connectivity — the threshold that wasn't fully resolved.** `is_hub_connected` is currently defined as *"a hub exists within the H3 k-ring(1) search radius"* — a presence check, not an explicit distance threshold. This has a real consequence: if the search radius were widened later to convert more of the 59 null-distance zones into real values (Option B, considered and explicitly rejected), `is_hub_connected` would become trivially true for almost everyone, unless it were redefined as a genuine distance cutoff (a candidate value of 1.0km was proposed but never finalized). **The decision taken** was Option A — leave the search radius as-is, and treat the 59 nulls as a real "no hub nearby" signal rather than as missing data to be filled in. This is documented directly in a markdown cell in `04_gold_layer.ipynb`, on the theory that an undocumented judgment call is worse than an imperfect one.

## 7. Power BI / DAX Patterns

**Live DirectQuery via Databricks SQL Warehouse (Import mode).** Rather than a static CSV export, Power BI connects directly to the Databricks SQL Warehouse's Server Hostname + HTTP Path, authenticated with a Personal Access Token, loaded in Import mode (data cached in the model, not queried live per-visual). This mirrors how a production BI deployment would actually be wired up, and it means the Gold layer can change without re-exporting anything.

**The Top-N grain gotcha.** Filtering a table visual to "top 10" only works correctly if the field being filtered uniquely identifies the thing you're ranking. Filtering by a zone name field to find the "top 10 worst routes" silently ranks by *origin zone* (aggregating across every destination that zone has), not by route. Fix: a DAX calculated column concatenating `origin_loc_id` and `dest_loc_id` into a synthetic `Route` key, used as the Top-N filter's target field instead.

**Scatter chart aggregation.** A scatter chart with no field in its "Details" bucket collapses every row into a single aggregated point. To plot one bubble per O-D pair, `origin_loc_id` and `dest_loc_id` both had to be added to Details explicitly — an easy step to miss, since the chart still renders (just wrong) without it.

**DirectQuery's `Sort by Column` license restriction.** The normal fix for "day names sorting alphabetically instead of chronologically" — a `Sort by Column` property pointing at a helper integer column — hit a capacity/license restriction specific to this DirectQuery connection. The workaround: instead of a numeric sort-order column, build a **text label column with a zero-padded numeric prefix** (`"01 - April"`, `"02 - May"`, ..., `"1 - Monday"`, `"2 - Tuesday"`, ...). Plain alphabetical text sort — Power BI's default, no special configuration — then produces the correct chronological order for free.

**Working around a blocked visual.** The Azure Maps visual requires Azure AD authentication that the available organizational email couldn't satisfy. Rather than blocking on that, the classic Map visual (Bing-based, flagged for retirement but still functional) was used instead, with a DAX `SWITCH`-based `Distance Band` categorical column (Near/Medium/Far) fed into the Legend field — approximating the color-gradient effect Azure Maps would have given natively via a continuous "Color saturation" field.


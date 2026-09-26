# Dataset

## Source

The dataset — `Urban_Flow_Analytics_Taxi_Dataset` — was provided as part of the **DATATHON 2026** competition, distributed as **12 monthly CSV files** (April 2025 through March 2026) via a shared Google Drive folder, totaling roughly **5 GB**.

> **Note:** if the organizers issued a formal citation or license for the underlying data, add it here. As received, the files carry no explicit license — this repo treats them as competition-provided data, not as redistributable open data.

The files were pulled directly from Google Drive into a Databricks Unity Catalog volume (`workspace.bronze.raw_files`) using `gdown.download_folder`, bypassing a local download/re-upload step.

## What's in it

Each monthly CSV is a trip-level taxi record, structured similarly to the NYC TLC trip-record format:

- Pickup / dropoff timestamps
- Pickup / dropoff location IDs (`pu_loc_id`, `do_loc_id`) and boroughs
- Trip distance, duration, average speed (pre-computed: `trip_duration_min`, `avg_speed_mph`)
- `provider_code` — identifies the technology/dispatch company operating the trip. **There is no vehicle identifier in the raw data** — this became a design constraint (see [TECHNIQUES.md](TECHNIQUES.md) for how a synthetic `vehicle_id` was derived)

A separate **zone lookup table** (`dim_zone`) provides, per `loc_id`:
`borough_name`, `zone_name`, `service_zone`, `centroid_lat`, `centroid_lon`.

## Scale, by layer

| Table | Layer | Rows | Notes |
|---|---|---|---|
| `raw_files` (12 CSVs) | Bronze | ~5 GB total | untouched source |
| `trips_clean` | Silver | — | after `is_valid_trip` filtering |
| `trips_enriched` | Silver | — | + hub-connectivity + synthetic `vehicle_id` |
| `dim_zone` | Gold | 265 | 5 rows with null centroid (see below) |
| `dim_provider` | Gold | 4 | no nulls |
| `dim_time_bucket` | Gold | 24 | no nulls |
| `dim_transit_hub` | Gold | 445 | no nulls |
| `dim_date` | Gold | 365 | 2025-04-01 → 2026-03-31, no nulls |
| `fact_od_congestion` | Gold | 45,767 | one row per origin–destination pair |
| `fact_vehicle_daily` | Gold | 1,692,882 | vehicle-day grain |
| `fact_zone_hub_connectivity` | Gold | 258 | 59 rows (22.87%) with null distance |

## Known data-quality issues (documented, not hidden)

**1. Corrupt timestamps that slipped past the first validity filter.**
The initial `is_valid_trip` filter (duration, distance, speed, lat/lon bounds) did not check date plausibility. `pickup_date` values ranged from 2008-12-31 to 2026-04-01, even though the true dataset scope is the 12 months of April 2025–March 2026. **Fix:** a date-range plausibility check (2025-04-01 to 2026-03-31) was added directly into `is_valid_trip`'s definition, and the whole downstream pipeline (Silver → Gold) was cascade-rebuilt. `dim_date` dropped from a suspicious 6,301 rows to the correct 365, and `fact_vehicle_daily` dropped by 4 rows (outlier vehicle-days removed).

**2. Five zones in `dim_zone` have null centroid coordinates.**
- `loc_id` 264 and 265 (`Unknown`, `N/A / Outside of NYC`) — standard TLC placeholder zones, expected.
- `loc_id` 104 and 105 (both variants of `Governor's Island/Ellis Island/Liberty Island`) — ferry-only islands with no fixed transit infrastructure, plausibly excluded from the geocoding source.
- `loc_id` 57 (`Corona`, Queens) — a genuinely populated zone with no obvious reason to be missing centroid data. **Left as a documented, unresolved gap** rather than imputed.

**3. 59 of 258 zones (22.87%) have no matching transit hub within the search radius.**
Breakdown by borough: Queens 34, Staten Island 11, Bronx 7, Brooklyn 4, Manhattan 2, EWR 1. This is not random noise — Queens and Staten Island are known to have sparser subway coverage than Manhattan. **Decision:** these nulls are kept as a meaningful "no hub found nearby" signal (rather than widening the H3 k-ring search radius to force a match), consistent with the borough pattern. See [TECHNIQUES.md](TECHNIQUES.md) for the full reasoning and the trade-off this created for the `is_hub_connected` flag.


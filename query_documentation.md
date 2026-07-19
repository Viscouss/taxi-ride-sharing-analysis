# Query Documentation

This project uses four Databricks SQL notebooks to explore whether NYC yellow taxi trips could be pooled into shared rides, and whether that opportunity changes on rainy vs. dry days. All four queries share the same two input tables:

- **`yellow_tripdata_january` / `_february` / `_march`** — NYC TLC yellow taxi trip records (see `data_dictionary_trip_records_yellow.pdf`).
- **`ny-weather-data`** — NOAA daily weather observations for the Central Park station (see `data_dictionary_weather.md`).

They're designed to be read in sequence: each one reuses the same cleaning and rain-flagging logic from the previous file, then asks a progressively more specific question about ride-sharing potential.

---

## 1. `rainy_vs_dry_day_overview.dbquery.ipynb` — Rainy vs. Dry Day Trip Overview

**Question answered:** Do overall taxi trip volumes and characteristics differ between rainy and dry days?

**What it does:**
1. `all_trips` — stacks the three monthly trip tables (Jan/Feb/Mar) into one dataset with `UNION ALL`.
2. `weather_flags` — reads the weather table and derives `is_rainy_day`: a day is "rainy" if precipitation exceeds 25 tenths-mm (2.5mm) **or** the `WT16` (rain) flag is set. Also converts `TMAX`/`TMIN` from tenths-of-°C to whole Celsius.
3. `clean_trips` — inner-joins each trip to its pickup day's weather record and filters out bad data: non-positive fares/totals, zero-distance trips (GPS errors or cancellations), and passenger counts outside 1–6 (data entry errors).
4. Final `SELECT` — groups by rainy/dry and reports, per weather type: number of distinct days, total trips, average trips per day, average fare, average trip distance, and average passengers per trip.

**Output:** A two-row summary table (Rainy Day vs. Dry Day) — the baseline comparison that motivates the rest of the analysis.

---

## 2. `shareable_ride_groups_by_weather.dbquery.ipynb` — Shareable Ride Groups by Weather

**Question answered:** How many trips could plausibly have been pooled into a shared ride, and does that count change with weather?

**What it does:**
1. Rebuilds `all_trips`, `weather_flags`, and `clean_trips` the same way as query 1 (weather flag logic unchanged), but drops the temperature fields since they aren't needed here.
2. Adds `time_bucket_4min` — snaps each pickup timestamp down to a fixed time window using `FLOOR(UNIX_TIMESTAMP(...) / 240) * 240` (240 seconds = 4 minutes), then converts back to a timestamp with `TIMESTAMP_SECONDS()`.
   > Note: the column is named `time_bucket_4min` and the code comment says "15-minute slot," but the divisor (240) actually buckets into 4-minute windows — the naming/comment is inconsistent with the logic and worth reconciling before presenting results.
3. `shareable_groups` — groups cleaned trips by pickup date, rain flag, pickup zone (`PULocationID`), dropoff zone (`DOLocationID`), and time bucket. A group represents trips that started from the same zone, ended in the same zone, within the same short time window — i.e., candidates for pooling into one shared vehicle. `HAVING COUNT(*) >= 2` keeps only groups where at least two trips could have shared a ride.
4. Final `SELECT` — groups by rainy/dry and reports: number of shareable groups found, total trips involved in those groups, average group size, largest group found, a breakdown of group counts by size (exactly 2, exactly 3, 4+), and the average fare within shareable trips.

**Output:** A two-row summary (Rainy Day vs. Dry Day) quantifying the scale of the ride-pooling opportunity under each weather condition.

---

## 3. `top_routes_for_ridesharing.dbquery.ipynb` — Top Routes for Ride-Sharing

**Question answered:** Which specific pickup → dropoff routes offer the most ride-sharing opportunities, and how does weather affect each?

**What it does:**
1. Same `all_trips` → `weather_flags` → `clean_trips` → `time_bucket_4min` pipeline as query 2.
2. `shareable_groups` — same grouping/`HAVING COUNT(*) >= 2` logic as query 2, but this time keeps `PULocationID` and `DOLocationID` as standalone output columns (rather than only using them as grouping keys) alongside `trips_in_group` and `avg_fare`.
3. Final `SELECT` — groups by pickup zone, dropoff zone, and rain flag, then reports for each route/weather combination: how many separate shareable groups occurred on that route, total trips on that route, and average fare. Sorted by `times_this_route_was_shareable` descending, limited to the **top 15 routes**.

**Output:** A ranked list of the 15 pickup/dropoff zone pairs with the most ride-sharing opportunities, split by rainy vs. dry weather — useful for identifying where to prioritize a pooling feature (e.g., specific airport or business-district routes).

---

## 4. `estimated_rider_savings.dbquery.ipynb` — Estimated Rider Savings from Pooling

**Question answered:** If shareable trips were actually pooled, how much money would riders save, and how does that total change with weather?

**What it does:**
1. Same `all_trips` → `weather_flags` → `clean_trips` → shareable-group pipeline as queries 2–3 (variable is named `time_bucket_15min` here despite still using the 240-second/4-minute bucket logic — same naming inconsistency as query 2).
2. `shareable_groups` — same grouping and `HAVING COUNT(*) >= 2` filter, aggregating `trips_in_group` and `avg_total_per_trip` (average fare paid per trip in the group).
3. `savings_calculated` — applies a **tiered discount model** to each shareable group based on its size:
   - **Pairs (2 riders):** each pays 60% of the solo fare → saves 40%.
   - **Trios (3 riders):** each pays 45% of the solo fare → saves 55%.
   - **4+ riders:** each pays 35% of the solo fare → saves 65%.
   
   It computes, per group: solo cost per person (the unmodified average fare), shared cost per person, per-person dollar savings, and total group savings (per-person savings × number of people in the group).
4. Final `SELECT` — groups by rainy/dry and reports: number of shareable groups, total trips involved, average solo fare, average shared fare, average savings per person, average savings percentage, total potential savings across the dataset, average daily savings (the headline business-case number), and a breakdown of total savings contributed by pairs, trios, and groups of 4+.

**Output:** A two-row summary (Rainy Day vs. Dry Day) translating the ride-pooling opportunity into an estimated dollar-savings business case, showing whether rainy days produce more or less pooling value than dry days.

---

## Shared logic across all four queries

| Element | Definition |
|---|---|
| `is_rainy_day` | `1` if daily precipitation (`PRCP`) > 25 tenths-mm (2.5mm) **or** the `WT16` rain flag is set; `0` otherwise. |
| Data cleaning filters | `fare_amount > 0`, `total_amount > 0`, `trip_distance > 0`, `passenger_count` between 1 and 6 inclusive. |
| Join key | `DATE(tpep_pickup_datetime)` (taxi) = `DATE` (weather), i.e., trips are matched to the weather of their **pickup day**. |
| "Shareable group" | 2 or more trips with the same pickup zone, same dropoff zone, and pickup times within the same time bucket. |

## Known inconsistencies worth cleaning up

- Queries 2 and 4 name their time-bucket column `time_bucket_4min` / `time_bucket_15min` respectively, and query 2's inline comment describes "15-minute slot" logic — but all three queries (2, 3, 4) use the same `/ 240` divisor, which is actually a **4-minute** bucket. Query 1's comment (`900 = 15 minutes × 60 seconds`) also doesn't match the `240` used in the code. Worth confirming which bucket width was intended (4 vs. 15 minutes) since it materially changes how "shareable" is defined, then aligning the column names/comments across all four files.

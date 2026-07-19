# Taxi Ride-Sharing Analysis

Analysis of NYC yellow taxi trip data (Q1: January–March) joined with NOAA daily weather data, exploring how many trips could plausibly have been pooled into shared rides, where that opportunity is concentrated, how much riders could save, and whether rainy days change any of it.

## Data sources

| File | Description |
|---|---|
| `data_dictionary_trip_records_yellow.pdf` | Official NYC TLC data dictionary for the yellow taxi trip record tables (`yellow_tripdata_january/february/march`). |
| `data_dictionary_weather.md` | Data dictionary for the weather dataset, written for this project (see below). |
| `weather_data_sample_2026_06_24_11_33_53.csv` | Sample of the raw NOAA GHCN-Daily weather feed (Central Park station) used to build the data dictionary. |

The taxi trip tables and the full weather table (`ny-weather-data`) referenced by the queries live in Databricks and are not included in this repo — only a data sample and the trip data dictionary are checked in for reference.

## Queries

Four Databricks SQL notebooks, meant to be read in order — each builds on the same cleaning/rain-flagging logic and asks a progressively more specific question. Full breakdown of each one, including the CTEs and known caveats, is in [`query_documentation.md`](query_documentation.md).

| Notebook | Question |
|---|---|
| `rainy_vs_dry_day_overview.dbquery.ipynb` | Do trip volume/fare/distance differ between rainy and dry days? |
| `shareable_ride_groups_by_weather.dbquery.ipynb` | How many trips could have been pooled, and does that change with weather? |
| `top_routes_for_ridesharing.dbquery.ipynb` | Which pickup → dropoff routes have the most pooling opportunity? |
| `estimated_rider_savings.dbquery.ipynb` | If pooled, how much would riders save, by weather? |

All four join taxi trips to weather on `DATE(tpep_pickup_datetime) = DATE`, flag a day as rainy when `PRCP > 25` (2.5mm) or the `WT16` rain flag is set, and define a "shareable group" as 2+ trips sharing a pickup zone, dropoff zone, and pickup time bucket.

## Known issue

The time-bucket logic across queries 2–4 is inconsistently named/commented (`time_bucket_4min` vs. `time_bucket_15min`, comments mentioning 15 minutes while the code uses a 4-minute divisor). See the "Known inconsistencies" section of `query_documentation.md` before relying on the bucket width for downstream decisions.

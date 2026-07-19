# Data Dictionary — Weather Dataset

**Source file:** `weather_data_sample_2026_06_24_11_33_53.csv`
**Provider:** NOAA National Centers for Environmental Information (NCEI) — Global Historical Climatology Network Daily (GHCN-Daily)
**Station in sample:** `USW00094728` — NY CITY CENTRAL PARK, NY US (Lat 40.77898, Lon -73.96925, Elevation 42.7 m)
**Grain:** One row per station per calendar day.

This dataset is joined to the taxi trip data on `DATE` (weather day) = `DATE(tpep_pickup_datetime)` (taxi pickup day) in all four queries. Only a handful of fields are actually used by the queries (`DATE`, `PRCP`, `TMAX`, `TMIN`, `WT16`) — the rest are included here for completeness since they're present in the raw feed.

## Column naming pattern

Every measured element has a paired `<ELEMENT>_ATTRIBUTES` column. The attributes column is a comma-delimited string of four flags in this order:

`MEASUREMENT FLAG, QUALITY FLAG, SOURCE FLAG, TIME OF OBSERVATION`

- **Measurement flag** — notes about how the value was measured (e.g. `T` = trace amount of precipitation too small to measure).
- **Quality flag** — non-blank means the value failed a NOAA quality check and should be treated with caution.
- **Source flag** — which NOAA network/source produced the reading.
- **Time of observation** — hour-minute (e.g. `2400`) the reading was taken, when applicable.

An empty attributes string (`,,1,2400`-style with blanks) generally means the value passed quality checks and needs no special caveat.

## Station identity columns

| Column | Description |
|---|---|
| `STATION` | NOAA station identifier (GHCN ID). One value in this sample: `USW00094728`. |
| `DATE` | Observation date, `YYYY-MM-DD`. This is the join key to taxi pickup date. |
| `LATITUDE` | Station latitude, decimal degrees. |
| `LONGITUDE` | Station longitude, decimal degrees. |
| `ELEVATION` | Station elevation, meters. |
| `NAME` | Station name and location, e.g. `NY CITY CENTRAL PARK, NY US`. |

## Core measurements (used in the SQL queries)

| Column | Description | Units |
|---|---|---|
| `PRCP` | Total precipitation for the day. | Tenths of mm (divide by 10 for mm) |
| `TMAX` | Maximum temperature for the day. | Tenths of °C (divide by 10 for °C) |
| `TMIN` | Minimum temperature for the day. | Tenths of °C (divide by 10 for °C) |
| `WT16` | Weather type flag: **Rain** occurred that day (may include freezing rain/drizzle). `1` = occurred, blank/null = did not occur or not reported. | Flag (1 or null) |

> The queries derive `is_rainy_day` as `PRCP > 25` (i.e. more than 2.5mm of rain) **OR** `WT16 = 1`, treating null `WT16` as 0 via `COALESCE`.

## Other precipitation / snow measurements

| Column | Description | Units |
|---|---|---|
| `SNOW` | Snowfall for the day. | mm |
| `SNWD` | Snow depth at time of observation. | mm |
| `WESD` | Water equivalent of snow on the ground. | Tenths of mm |
| `EVAP` | Evaporation from an evaporation pan. | Tenths of mm |
| `MDEV` / `DAEV` | Multiday evaporation total, and number of days it covers (used together when a station doesn't report daily). | Tenths of mm / days |
| `MDSF` / `DASF` | Multiday snowfall total, and number of days it covers. | mm / days |

## Temperature, humidity, and pressure

| Column | Description | Units |
|---|---|---|
| `TAVG` | Average temperature for the day (as reported by station; not always `(TMAX+TMIN)/2`). | Tenths of °C |
| `TOBS` | Temperature at the time of observation. | Tenths of °C |
| `ADPT` | Average dew point temperature. | Tenths of °C |
| `AWBT` | Average wet bulb temperature. | Tenths of °C |
| `RHAV` | Average relative humidity for the day. | Percent |
| `RHMN` | Minimum relative humidity for the day. | Percent |
| `RHMX` | Maximum relative humidity for the day. | Percent |
| `ASLP` | Average sea level pressure. | hPa (tenths) |
| `ASTP` | Average station pressure. | hPa (tenths) |

## Wind

| Column | Description | Units |
|---|---|---|
| `AWND` | Average daily wind speed. | Tenths of m/s |
| `WDMV` | 24-hour wind movement (total distance of wind passing the station). | km |
| `WSF1` / `WDF1` | Fastest 1-minute wind speed / direction. | Tenths of m/s / degrees |
| `WSF2` / `WDF2` | Fastest 2-minute wind speed / direction. | Tenths of m/s / degrees |
| `WSF5` / `WDF5` | Fastest 5-second wind speed / direction. | Tenths of m/s / degrees |
| `WSFG` / `WDFG` | Peak wind gust speed / direction. | Tenths of m/s / degrees |
| `WSFM` / `WDFM` | Fastest monthly wind speed / direction. | Tenths of m/s / degrees |
| `MDWM` / `DAWM` | Multiday wind movement total, and number of days it covers. | km / days |
| `FMTM` | Time of fastest 1-minute wind. | HHMM |
| `PGTM` | Time of peak wind gust. | HHMM |

## Sky cover / sunshine

| Column | Description | Units |
|---|---|---|
| `ACMH` | Average cloudiness, midnight to midnight, manual observation. | Percent |
| `ACSH` | Average cloudiness, sunrise to sunset, manual observation. | Percent |
| `PSUN` | Percent of possible daily sunshine. | Percent |
| `TSUN` | Total sunshine for the day. | Minutes |

## Weather type flags (`WT01`–`WT22`)

Each is a boolean indicator: `1` = this weather type was observed at the station that day, blank/null = not observed or not applicable. These come from the daily summary of present weather (WT) codes.

| Column | Weather type |
|---|---|
| `WT01` | Fog, ice fog, or freezing fog |
| `WT02` | Heavy fog or heaving freezing fog |
| `WT03` | Thunder |
| `WT04` | Ice pellets, sleet, snow pellets, or small hail |
| `WT05` | Hail |
| `WT06` | Glaze or rime |
| `WT07` | Dust, volcanic ash, blowing dust/sand, or blowing obstruction |
| `WT08` | Smoke or haze |
| `WT09` | Blowing or drifting snow |
| `WT11` | High or damaging winds |
| `WT13` | Mist |
| `WT14` | Drizzle |
| `WT15` | Freezing drizzle |
| `WT16` | Rain (may include freezing rain, drizzle, and freezing drizzle) — **used to flag rainy days** |
| `WT17` | Freezing rain |
| `WT18` | Snow, snow pellets, snow grains, or ice crystals |
| `WT19` | Unknown source of precipitation |
| `WT21` | Ground fog |
| `WT22` | Ice fog or heavy freezing fog |

## Notes / caveats for analysis

- Values of `null` are common for many of the secondary columns (humidity, pressure, sunshine, etc.) — Central Park's manual station does not report every element daily. `PRCP`, `TMAX`, `TMIN`, and the `WT` flags are the most consistently populated and are the only ones the current queries rely on.
- All temperature and precipitation values are stored in **tenths** of their natural unit (tenths of °C, tenths of mm) — always divide by 10 before presenting to end users, as the queries already do for temperature (`TMAX / 10.0`).
- `PRCP` of `0` with a `T` measurement flag in `PRCP_ATTRIBUTES` means "trace" precipitation (a non-zero but unmeasurably small amount), not literally zero.

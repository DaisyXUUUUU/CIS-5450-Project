# Data dictionary: `trips_clean_v1.parquet`

One row per sampled credit-card yellow taxi trip after cleaning. Produced by `notebooks/01_data_pipeline.ipynb`.

## Do not use as model features

| Column | Why |
| --- | --- |
| `tip_amount` | It is the outcome. |
| `total_amount` | Includes the tip. |
| `tip_pct` | The outcome as a ratio. |
| `high_tip`, `high_tip_20`, `high_tip_30` | Target labels. |
| `payment_type` | Always 1 (card) after filtering. |
| `tpep_dropoff_datetime` | Only used to compute `duration_min`. |
| `split`, `source_month`, `pickup_hour_ts` | Bookkeeping. |

## Columns

| Column | Type | Meaning |
| --- | --- | --- |
| `VendorID` | int | Meter provider (1 Creative Mobile, 2 Curb, 6 Myle, 7 Helix). Categorical. |
| `tpep_pickup_datetime` | datetime | Pickup time, New York local. Source for hour, day of week, weekend, late night. |
| `tpep_dropoff_datetime` | datetime | Drop-off time. Not a feature. |
| `passenger_count` | float/Int | Passengers. 0 was set to missing; impute in the pipeline on train only. |
| `trip_distance` | float | Metered miles, 0 < x <= 100. |
| `RatecodeID` | float/Int | 1 standard, 2 JFK, 3 Newark, 4 Nassau/Westchester, 5 negotiated, 6 group. 99 removed. Categorical. |
| `store_and_fwd_flag` | str | Y if the record was stored in the vehicle before upload. |
| `PULocationID`, `DOLocationID` | int | TLC taxi zone IDs (1 to 263 after cleaning). JFK = 132, LaGuardia = 138, Newark = 1. |
| `fare_amount` | float | Time and distance fare, > 0. Denominator of `tip_pct`. |
| `extra` | float | Miscellaneous extras and surcharges (rush hour, overnight). |
| `mta_tax` | float | MTA tax. |
| `tolls_amount` | float | Tolls. |
| `improvement_surcharge` | float | Improvement surcharge. |
| `congestion_surcharge` | float | New York State congestion surcharge. |
| `Airport_fee` | float | Fee for pickups at LaGuardia and JFK. |
| `cbd_congestion_fee` | float | Manhattan Central Business District congestion charge. |
| `source_month` | str | Month file the trip was sampled from. |
| `duration_min` | float | Drop-off minus pickup, in minutes, 1 to 180. |
| `PU_Borough`, `DO_Borough` | str | Borough from the zone lookup. |
| `PU_Zone`, `DO_Zone` | str | Zone name from the zone lookup. |
| `PU_service_zone`, `DO_service_zone` | str | Yellow Zone, Boro Zone, Airports, or EWR. |
| `pickup_hour_ts` | datetime | Pickup time floored to the hour; weather join key. Not a feature. |
| `temp_c` | float | Open-Meteo temperature (°C) at that hour. |
| `precip_mm` | float | Open-Meteo precipitation (mm) accumulated in the hour ending at `pickup_hour`. |
| `is_rain` | int | 1 if `precip_mm >= 0.1`. |
| `pickup_hour` | int | Hour of pickup, 0 to 23 (Table 1 no. 1). |
| `pickup_dow` | int | Day of week, 0 = Monday to 6 = Sunday (no. 2). |
| `is_weekend` | int | Pickup on Saturday or Sunday (no. 14). |
| `is_late_night` | int | Pickup 00:00 to 05:59 (no. 15, H1 group). |
| `avg_speed_mph` | float | 60 x `trip_distance` / `duration_min` (no. 16). Values > 80 mph are suspicious; cap or drop using train only. |
| `is_airport` | int | Pickup or drop-off at JFK (132) or LaGuardia (138) (no. 17, H3 group). |
| `is_newark` | int | Pickup or drop-off at Newark (1); excluded from H3. |
| `same_borough` | int | `PU_Borough == DO_Borough` (no. 18). |
| `has_cbd_fee` | int | `cbd_congestion_fee > 0` (no. 19). |
| `is_short_trip` | int | 0 < `trip_distance` <= 2 miles (H4 group). |
| `tip_pct` | float | `tip_amount / fare_amount`, 0 < x <= 1. Not a feature. |
| `high_tip` | int | 1 if `tip_pct >= 0.25`. Main target. |
| `high_tip_20`, `high_tip_30` | int | Same at 20% and 30%. Sensitivity checks only. |
| `pickup_date` | datetime | Pickup date; cluster unit for clustered SEs and the day-block bootstrap. |
| `split` | str | `train` (Apr 1 to May 15), `val` (May 16 to 31), `test` (June). |

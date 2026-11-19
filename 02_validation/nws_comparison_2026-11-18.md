# Provo NWS comparison

- **Run date:** 2026-11-18 (America/Denver)
- **Location:** Provo, UT, `40.2338, -111.6585`
- **NWS point lookup:** <https://api.weather.gov/points/40.2338,-111.6585>
- **NWS hourly forecast:** <https://api.weather.gov/gridpoints/SLC/105,151/forecast/hourly>
- **Open-Meteo forecast:** <https://api.open-meteo.com/v1/forecast?latitude=40.2338&longitude=-111.6585&hourly=temperature_2m,wind_speed_10m,wind_gusts_10m,precipitation_probability&temperature_unit=fahrenheit&wind_speed_unit=mph&precipitation_unit=inch&forecast_days=7&timezone=auto>
- **NWS generated:** `2026-11-19T05:15:33Z`; updated `2026-11-18T22:52:07Z` (`2026-11-18 15:52:07-07:00`)
- **Open-Meteo timezone:** `America/Denver`; response contained 168 hourly values over 2026-11-18 through 2026-11-24.
- **Alignment:** Matched by local date and hour in `America/Denver`. NWS began at 22:00 on Nov 18, so the 22 hours earlier that day had no matching NWS forecast; 146 hourly buckets were compared. Both APIs supplied matching temperature, wind, and precipitation values for all 146 buckets. NWS hourly gusts are not provided; gust comparison is unavailable.
- **Normalization:** NWS temperature values were converted from the period unit to °F (the returned periods were already °F). NWS `windSpeed` text was parsed to mph (the returned values were single mph values, not ranges). Precipitation probabilities were already percentages. Open-Meteo values were requested in °F, mph, and percent.
- **Result:** **FAIL — investigate before release.** Temperature: 33/146 pass at ±2°F, 113 fail; maximum absolute difference 10.7°F. Sustained wind: 102/146 pass at ±3 mph, 44 fail; maximum absolute difference 12.8 mph. Precipitation probability: 101/146 pass at ±10 percentage points, 45 fail; maximum absolute difference 52 points.
- **Investigation:** Verified the point endpoint resolved to the Provo/SLC forecast grid, both responses used `America/Denver` local-hour alignment, units were normalized as above, and all values were paired at identical local hours. The mismatches therefore remain forecast-source disagreements in this snapshot, not an observed timestamp or unit-conversion error. Keep this comparison as a release warning; do not silently change the app thresholds to force agreement. Re-run against fresh forecasts before release because both providers update independently.
- **Detailed log:** [nws_comparison_2026-11-18.csv](./nws_comparison_2026-11-18.csv) records each matched timestamp, raw and normalized source values, signed differences, per-field pass/fail, and gust availability.

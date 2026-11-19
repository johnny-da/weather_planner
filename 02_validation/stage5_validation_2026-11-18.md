# Stage 5 validation

## Forecast logic and browser checks

- **Provo hand fixture:** Day 1 inputs `32°F`, `15 mph` sustained wind, `22 mph` gust, and `10%` rain produced Crane `GO`, Concrete `NO-GO`, Roofing `GO`.
- **Threshold boundary matrix:** Values immediately below, at, and immediately above each listed threshold were exercised through mocked Open-Meteo responses. Every case matched the rule (`<` is below; `>=` enters the threshold category):

| Threshold checks | Values tested | Expected and observed statuses |
|---|---|---|
| Crane sustained Go / No-Go | 19.99, 20, 20.01, 24.99, 25, 25.01 mph | GO, CAUTION, CAUTION, CAUTION, NO-GO, NO-GO |
| Crane gust Go / No-Go | 24.99, 25, 25.01, 29.99, 30, 30.01 mph | GO, CAUTION, CAUTION, CAUTION, NO-GO, NO-GO |
| Concrete caution / Go minimum | 34.99, 35, 35.01, 39.99, 40, 40.01°F | NO-GO, CAUTION, CAUTION, CAUTION, GO, GO |
| Concrete rain No-Go | 59.99, 60, 60.01% | GO, NO-GO, NO-GO |
| Roofing rain Go / No-Go | 19.99, 20, 20.01, 39.99, 40, 40.01% | GO, CAUTION, CAUTION, CAUTION, NO-GO, NO-GO |
| Roofing gust Go / No-Go | 24.99, 25, 25.01, 29.99, 30, 30.01 mph | GO, CAUTION, CAUTION, CAUTION, NO-GO, NO-GO |

- **Incomplete data:** A day with all precipitation-probability values missing displayed `CAUTION` for concrete and roofing, with detail reasons naming missing maximum rain probability.
- **Coordinates/map:** Latitude `91` showed a field error and made no API request. A map click and marker drag updated coordinate fields; applying those locations fetched forecasts. `Reset to Provo` restored `40.2338`, `111.6585` and requested a forecast.
- **Threshold settings:** Default thresholds validate and apply. An equal Crane sustained Go/No-Go pair is rejected with a field error and leaves ratings unchanged; Reset to Defaults clears the error and restores `25 mph` No-Go.
- **API failure/recovery:** HTTP 503 retried once and succeeded on request 2. HTTP 400 was not retried, showed the retry control, and removed forecast cards. After a successful load followed by HTTP 400, `Use last loaded forecast` restored the seven Provo cards without another request.
- **Details/raw data:** Keyboard activation opened the selected day, showed the daily metrics and exact activity reasons, and rendered all 24 hourly rows. Raw values retained decimal precision and nulls displayed as `—`; the copy control produced a tab-separated table with the header and 24 data rows.
- **Keyboard/focus:** The drawer receives initial focus at Close, Tab/Shift+Tab stay within the open drawer (including when raw details are collapsed), Escape closes it, and focus returns to the selected card.
- **Responsive widths:** At CSS widths 1440, 1024, and 390 px, the app had no horizontal overflow. Workspace columns were 2, 2, and 1; forecast columns were 7, 3, and 1; map heights were 420, 420, and 360 px. The day detail panel used the desktop drawer at 1440/1024 and mobile bottom sheet at 390.
- **Live app smoke check:** The browser loaded the actual Leaflet map/Provo popup and a live Open-Meteo response with 168 hours in `America/Denver`, rendering seven daily cards. Automated behavior checks used controlled API fixtures so boundary and failure results were repeatable.

## National Weather Service comparison

The official NWS comparison was performed against a live Open-Meteo response. **It does not meet the specified tolerances**: 113/146 temperature hours, 44/146 sustained-wind hours, and 45/146 precipitation-probability hours were out of tolerance. The matching timestamps and complete per-hour values are recorded in [nws_comparison_2026-11-18.csv](./nws_comparison_2026-11-18.csv); source URLs, normalization, investigation, and release warning are in [nws_comparison_2026-11-18.md](./nws_comparison_2026-11-18.md). Re-run this comparison with fresh provider forecasts before release; do not treat the mismatch as a passing validation.

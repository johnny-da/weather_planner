# Jobsite Weather Planner

## Purpose
A single-page, field-ready web app for general contractors and project managers to evaluate 7-day weather forecasts for crane lifts, concrete pours, and roofing work. The app combines an interactive jobsite map with one Go/Caution/No-Go daily rating for each activity.

## Inputs
### Jobsite location
- Provide a Leaflet map with OpenStreetMap tiles and a single draggable marker.
- Clicking the map moves the marker and updates both coordinate fields. Dragging the marker also updates both fields.
- **Latitude:** number input labeled `Latitude (°N)`, range `-90.0000` to `90.0000`, step `0.0001`, default `40.2338`.
- **Longitude:** number input labeled `Longitude (°W)`, range `0.0000` to `180.0000`, step `0.0001`, default `111.6585`. Convert the positive west value to the API's signed longitude by applying a negative sign (`111.6585` becomes `-111.6585`).
- **Apply Location:** submitting blank, nonnumeric, out-of-range, or more-than-four-decimal-place coordinates shows a red field-specific error and performs no request. Valid coordinates move and recenter the marker, then fetch data.
- A request disables Apply Location and changes its label to `Loading...`. The marker starts at Provo, with zoom `11`.
- Place a small `Reset to Provo` button directly below or beside the map controls. When activated, set latitude to `40.2338`, set longitude UI value to `111.6585°W` (API value `-111.6585`), recenter the map at zoom `11`, open the `Provo, UT jobsite` popup, and fetch the Provo forecast. If a request is loading, disable the button or queue exactly one reset request after the active request finishes.

### Threshold settings
Tuck editable settings behind a collapsed-by-default `Threshold Settings` button labeled exactly that; it must not expose the threshold controls on the main screen until opened. Include number inputs with units, `Apply Thresholds`, and `Reset to Defaults` inside the panel.
- Crane sustained Go limit: `20 mph`, range `0–100`, step `1`; No-Go limit: `25 mph`, range `0–100`, step `1`; No-Go must be greater than Go.
- Crane gust Go limit: `25 mph`, range `0–150`, step `1`; No-Go limit: `30 mph`, range `0–150`, step `1`; No-Go must be greater than Go.
- Concrete Go minimum: `40°F`, range `-40` to `120`, step `1`; Caution minimum: `35°F`, range `-40` to `120`, step `1`; Caution must be lower than Go.
- Concrete No-Go rain probability: `60%`, range `0–100`, step `1`.
- Roofing Go rain probability: `20%`, range `0–100`, step `1`; No-Go rain probability: `40%`, range `0–100`, step `1`; No-Go must be greater than Go.
- Roofing gust Go limit: `25 mph`, range `0–150`, step `1`; No-Go limit: `30 mph`, range `0–150`, step `1`; No-Go must be greater than Go.
- Blank, nonnumeric, out-of-range, or conflicting values show inline errors and leave the existing thresholds unchanged. Reset restores all defaults and immediately recalculates the current forecast.

## Calculations
### API request
Fetch with this exact endpoint pattern:

`https://api.open-meteo.com/v1/forecast?latitude={LAT}&longitude={LON}&hourly=temperature_2m,wind_speed_10m,wind_gusts_10m,precipitation_probability&temperature_unit=fahrenheit&wind_speed_unit=mph&precipitation_unit=inch&forecast_days=7&timezone=auto`

Replace `{LAT}` and `{LON}` with signed decimal coordinates. Return values are °F, mph, and percent; `precipitation_unit=inch` is included for the US-customary configuration. Group hours by the local calendar date returned in `timezone=auto`. Do not substitute point forecasts or a weather-code-only request.

### Hourly-to-daily summaries
- Crane: maximum `wind_speed_10m` and maximum `wind_gusts_10m` for the local calendar day.
- Concrete: minimum `temperature_2m` and maximum `precipitation_probability` for the local calendar day.
- Roofing: maximum `precipitation_probability` and maximum `wind_gusts_10m` for the local calendar day.
- Omit null or nonnumeric hourly values from each statistic. A day missing a statistic needed by an activity is marked Caution with `Data incomplete`; it is never rated Go or No-Go. A normal response should contain 168 hourly records.

### Boundary and precedence rules
Compare parsed values exactly; equality follows the operator shown, with no unclassified value.
- **Crane:** No-Go if max sustained `>= sustained No-Go limit` OR max gust `>= gust No-Go limit`. Otherwise Caution if max sustained `>= sustained Go limit` OR max gust `>= gust Go limit`. Otherwise Go. Default: sustained `<20` and gust `<25` is Go; sustained `20<=wind<25` or gust `25<=gust<30` is Caution; sustained `>=25` or gust `>=30` is No-Go.
- **Crane source:** OSHA 29 CFR 1926.1400 requires employers to implement a wind plan, account for manufacturer prohibitions, and stop or modify crane operations under adverse conditions. Cite [29 CFR 1926.1400(a)](https://www.osha.gov/laws-regs/regulations/standardnumber/1926/1926.1400) and [29 CFR 1926.1400(c)(1)](https://www.osha.gov/laws-regs/regulations/standardnumber/1926/1926.1400). The `20/25/30 mph` defaults are project controls and are never a substitute for the crane's rated operating manual. Manufacturer controls supersede app defaults.
- **Concrete:** No-Go if minimum temperature `< Caution minimum` OR max rain probability `>= No-Go rain limit`. Otherwise Caution if minimum temperature `< Go minimum`. Otherwise Go. Default: temp `<35°F` or rain `>=60%` is No-Go; `35<=temp<40°F` is Caution; temp `>=40°F` with rain `<60%` is Go.
- **Concrete source:** [ACI 306R-16, Guide to Cold Weather Concreting](https://www.concrete.org/store/productdetail.aspx?ItemID=306R16&Language=English) defines cold-weather concreting precautions and discusses placement, protection, curing, and strength at temperatures below `40°F`. ACI Sections 3 and 6 support the `40°F` threshold and `35°F` protective lower boundary. The `60%` rain rule is an explicit project control, not an ACI threshold.
- **Roofing:** No-Go if max gust `>= gust No-Go limit` OR max rain probability `>= rain No-Go limit`. Otherwise Caution if max gust `>= gust Go limit` OR max rain probability `>= rain Go limit`. Otherwise Go. Default: rain `<20%` and gust `<25 mph` is Go; `20<=rain<40%` or `25<=gust<30 mph` is Caution; rain `>=40%` or gust `>=30 mph` is No-Go.
- **Roofing source:** use the [NRCA Roofing Manual](https://www.nrca.net/technical/guidelines) and current product-maker instructions. The [CertainTeed asphalt-shingle installation instructions](https://www.certainteed.com/resources/roofing-installation-instructions) illustrate manufacturer controls. The `20%/40%` rain and `25/30 mph` gust defaults are conservative project controls, not universal limits; the applicable roof-system manual supersedes them.

### Error handling
- Use a 12-second API timeout and retry once after network or 5xx failure; never retry 4xx responses.
- On timeout, network failure, HTTP error, invalid JSON, or missing required hourly fields, remove old forecast cards and show `Weather unavailable — check your connection and try again.` plus `Retry`.
- Preserve one most-recent successful in-memory response. When available, add `Use last loaded forecast`; it restores that response and shows `Using cached forecast — may be outdated`. Data does not persist across reload.
- Never convert an HTTP/API failure into a Go rating. Missing individual statistic values are Caution for the affected activity; no response produces no forecast cards.

## Outputs
### Leaflet map
- Render a 380 px desktop map panel with 420 px minimum map height and OpenStreetMap attribution visible. Clicking or dragging the marker updates the coordinate fields; marker movement is also announced through the status text.
- Initial view is Provo, UT: `40.2338`, `-111.6585`, zoom `11`, marker popup `Provo, UT jobsite`.

### Seven-day grid
- Above the grid show `7-Day Activity Forecast` and the location name or signed-coordinate fallback.
- Desktop `>=1200px`: seven equal columns. Tablet `769–1199px`: three columns. Mobile `<769px`: one column.
- Every card shows weekday and local date plus exactly one badge for Crane Lifts, Concrete Pours, and Roofing. Go: `GO`, `#DCFCE7` background, `#166534` text, `#86EFAC` border. Caution: `CAUTION`, `#FEF3C7`, `#92400E`, `#FCD34D`. No-Go: `NO-GO`, `#FEE2E2`, `#991B1B`, `#FCA5A5`. Badges use 12 px uppercase 800-weight labels.
- The whole card is keyboard- and pointer-accessible with `aria-label="View details for <weekday, month day>"`.

### Day detail panel
- A card click or Enter/Space opens a right-side drawer at `>=769px` and a bottom sheet below `769px`. The title is `<weekday, Month D, YYYY>`; the close control is labeled `Close details`.
- Show the final badge and exact reason for all three activities:
  - Crane: `Sustained wind: X mph; gust: Y mph. <rating> — <both below limits / exact crossed sustained limit / exact crossed gust limit>`.
  - Concrete: `Minimum temperature: X°F; maximum rain probability: Y%. <rating> — <minimum is below Z°F / rain is at or above Z% / exact Caution boundary / both limits are met>`.
  - Roofing: `Maximum rain probability: X%; maximum gust: Y mph. <rating> — <both below limits / exact rain boundary / exact gust boundary>`.
- The reason must name all thresholds actually crossed and may not replace them with generic advice. Incomplete data states `Data incomplete — Caution` and names the missing statistic.
- Default-closed `Raw Hourly Forecast` section contains a scrollable table: local time, `temperature_2m (°F)`, `wind_speed_10m (mph)`, `wind_gusts_10m (mph)`, and `precipitation_probability (%)`. Render null/missing as `—`, preserve source precision, and provide `Copy table` for tab-separated values.

## Look and feel
- Page background `#FFFFFF`; card/map canvas `#F3F4F6`; body text `#111827`; muted text `#6B7280`; primary navy `#1E3A5F`; focus/link blue `#2563EB`.
- Desktop `>=1200px`: two columns, fixed `380px` left map and fluid right forecast, `24px` outer gutter and `20px` gap. At `769–1199px`: fixed `340px` left and fluid right. Mobile `<769px`: stack map above forecast, `360px` map height, `12px` gutter.
- Cards use 12 px radius and `#D1D5DB` 1 px border; inputs use `#CBD5E1` 1 px border and 4 px radius. Use 8 px spacing increments, 12 px minimum body text, 16 px minimum control text, and 2 px `#2563EB` keyboard focus rings.
- Keep status badge colors restricted to the exact Output badge hex values. Hover/focus feedback may strengthen borders or translate 3 px, with no layout shift.
- Initial state: Provo, UT, latitude `40.2338` and longitude UI value `111.6585°W` (API signed value `-111.6585`), zoom `11`, marker popup open, a visible small `Reset to Provo` button beside or below the map controls, the `Threshold Settings` panel collapsed, default thresholds loaded, and an automatic Open-Meteo request.

## Validation
- Required location fixture: Provo, UT (`40.2338° N, 111.6585° W`) is both initial state and the hand-test location.
- **Hand-calculation test:** Forecasted Day 1: minimum temperature `32°F`, maximum sustained wind `15 mph`, maximum gust `22 mph`, maximum rain probability `10%`.
  - Crane Lifts: Go (`15 < 20 mph`, `22 < 25 mph`).
  - Concrete Pours: No-Go (`32°F < 35°F`).
  - Roofing: Go (`10% < 20%`, `22 mph < 25 mph`).
- Automated tests must keep this fixture and add exact-boundary checks immediately below, exactly at, and immediately above `20 mph`, `25 mph`, `30 mph`, `35°F`, `40°F`, `60%`, `20%`, and `40%`. No threshold boundary may fall through or produce contradictory ratings.
- **National Weather Service comparison (required):** for Provo, first call `https://api.weather.gov/points/40.2338,-111.6585`, follow `properties.forecastHourly`, and compare the first seven local calendar days with the app's Open-Meteo raw table. Normalize NWS temperature and wind text to °F and mph. A match passes when temperature is within `±2°F`, sustained wind within `±3 mph`, and precipitation probability within `±10 percentage points` for matching local-hour buckets. Gust is `Not available` unless both sources provide a matching-hour gust. Log date/time, source values, normalized values, differences, pass/fail, and both URLs in `02_validation/`; investigate any out-of-tolerance result before release.
- Test at `1440px`, `1024px`, and `390px` widths. Verify marker click/drag, coordinate errors, threshold errors, card keyboard activation, detail panel, raw table, API retry/cache behavior, and NWS comparison.

## Out of scope
- User accounts, authentication, profiles, cross-device settings, or account-based sharing.
- Historical/archive weather lookup or forecasts beyond the API's seven returned local calendar days.
- Saving, sharing, exporting, or remotely storing multiple jobsites; no multi-site database or project management.
- Address search/geocoding, reverse-geocoding, routing, or more than one marker per view.
- Radar, satellite, severe-weather, tornado, hurricane, lightning, or live emergency-alert tracking.
- Real-time sensors, anemometers, worker tracking, dispatch, scheduling, cost estimation, or automated safety decisions.
- Concrete cure-state modeling, roofing surface-moisture sensing, crew safety compliance certification, or permit generation.
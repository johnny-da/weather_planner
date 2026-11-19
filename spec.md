# Jobsite Weather Planner

## App overview
This single-page weather planner helps construction teams assess the next seven local forecast days for crane lifts, concrete pours, and roofing at one jobsite. A Leaflet map and city/location search select the site; Open-Meteo hourly data is summarized into per-activity Go/Caution/No-Go ratings with exact threshold reasons and raw values for verification. The page starts at Provo, UT and is designed for desktop and mobile use.

## Purpose
A single-page, field-ready web app for general contractors and project managers to evaluate 7-day weather forecasts for crane lifts, concrete pours, and roofing work. The app combines an interactive jobsite map with one Go/Caution/No-Go daily rating for each activity.

## Inputs
### Jobsite location
- Provide a Leaflet map with Esri World Street Map tiles, visible attribution, and a single draggable marker.
- Provide a text input labeled `Location search`, with placeholder `Search for a city or location (e.g., Provo, UT)`, and a `Search` button. Pressing Enter in the field submits the search. Trim whitespace, reject blank text without making a request, and cap input at 120 characters.
- Search using `https://geocoding-api.open-meteo.com/v1/search?name={URL_ENCODED_QUERY}&count=5&language=en&format=json`. If multiple results are returned, show up to five accessible choices labeled with city and state/region; include the country for non-US results where available. Selecting a result moves/recenters the marker, updates the location header, and loads the forecast. If exactly one result is returned, select it automatically. If none are returned, show `Location not found. Try a city and state, such as Provo, UT.` without moving the marker or replacing the current forecast. On service/network failure, show an explicit location-search error and keep the current forecast.
- Clicking the map or dragging the marker updates the location search field to `Map selection: {latitude}, {signed longitude}` using four decimal places and keeps the selected coordinates pending. Pressing `Search` then uses those coordinates directly rather than geocoding the display text; typing in the field cancels the pending map selection. This preserves manual map selection without requiring reverse geocoding.
- The marker starts at Provo (`40.2338, -111.6585`), with zoom `11`. A selected geocoding result uses its signed latitude/longitude directly in the forecast request.
- Place a small `Reset to Provo` button directly below or beside the map controls. When activated, set the search field to `Provo, UT`, recenter the map at zoom `11`, open the `Provo, UT jobsite` popup, and fetch the Provo forecast. If a request is loading, disable location controls until it finishes.

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
- Render a 380 px desktop map panel with 420 px minimum map height and Esri attribution visible. Clicking or dragging the marker updates the pending `Map selection: {latitude}, {signed longitude}` search value; marker movement is also announced through the status text. Selecting a search result or pressing Reset recenters the map and marker.
- Initial view is Provo, UT: `40.2338`, `-111.6585`, zoom `11`, marker popup `Provo, UT jobsite`.

### Seven-day grid
- Above the grid show `7-Day Activity Forecast` and the location name or signed-coordinate fallback.
- Place a `Copy Crew Brief` button in this heading row, beside the location pill on desktop and flowing with the heading on narrow screens. Enable it only when the current complete seven-day forecast is displayed; disable it while loading or when forecast data is unavailable or cleared.
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

## Features
### Feature 1: Best 3-Day Pour Window
- Evaluate every consecutive three-day stretch in the seven-day forecast (five overlapping windows). Use the existing daily Concrete Pours ratings and their underlying minimum temperatures and maximum precipitation probabilities.
- Rank fully evaluable windows. If one or more have zero `NO-GO` days, choose only among those; otherwise rank all fully evaluable windows and use the fallback below. Within the eligible candidates, rank lexicographically: first, most `GO` days wins; second, fewest `NO-GO` days wins. If still tied, compare each window's coldest-night minimum (the lowest of its three daily minimum temperatures; higher is better) and worst-day precipitation probability (the highest of its three daily maximum rain probabilities; lower is better). For each measure, calculate a 0–100 midrank percentile among the status-tied candidates: `100 × (number of strictly worse values + 0.5 × number of equal values) / number of candidates`. Average the temperature and rain percentiles with equal weight; highest combined rank wins. If still tied, choose the window that starts earliest. Show status counts so the weather tie-break never obscures the status-first ranking.
- Display a `Best 3-Day Pour Window` summary banner immediately above the seven-day cards. Include the local start and end dates, each day's concrete status, the Go/Caution/No-Go counts, the coldest-night minimum, the worst-day rain probability, and a concise reason for the ranking.
- Highlight the selected three cards with a consistent `Pour window` label or outline. Preserve each card's existing Go/Caution/No-Go badge colors and status; the highlight must not imply a status change. Keep the banner and card treatment responsive, stacking the banner content on mobile.
- If at least one fully evaluable window has zero No-Go days, highlight the highest-ranked such window. If every fully evaluable window contains one or more No-Go days, show the top-ranked result as `Best available — no fully evaluated 3-day window is No-Go-free`, state that no No-Go-free window could be confirmed, and show its daily No-Go reasons. Do not describe that fallback as ideal, recommended, or safe.
- Do not give missing forecast values favorable scores. Exclude windows with incomplete concrete inputs when at least one fully evaluable window exists and disclose that incomplete windows were excluded. If all five windows are incomplete, show `Unable to rank pour windows — forecast data incomplete` and do not highlight a candidate.

### Feature 2: Copy Crew Brief
- The `Copy Crew Brief` button copies a plain-text summary of the currently displayed forecast, suitable for pasting into a text message or email. Use the current location label, forecast timezone, dates, activity ratings, active thresholds, and Best 3-Day Pour Window result (including cached forecasts and threshold recalculations).
- Generate this exact structure, replacing placeholders with current values and including one daily block for each of the seven forecast dates:

  ```text
  JOBSITE WEATHER BRIEF — {location}
  Forecast: {first local date}–{last local date} ({timezone})

  BEST 3-DAY POUR WINDOW: {start date}–{end date}
  Concrete ratings: {Go count} Go, {Caution count} Caution, {No-Go count} No-Go
  Coldest overnight low: {temperature}°F · Worst daily rain probability: {probability}%
  {Include the best-available fallback statement when applicable.}

  {Weekday, Month D}
  Crane Lifts: {GO|CAUTION|NO-GO}{ — concise warning for CAUTION or NO-GO}
  Concrete Pours: {GO|CAUTION|NO-GO}{ — concise warning for CAUTION or NO-GO}
  Roofing: {GO|CAUTION|NO-GO}{ — concise warning for CAUTION or NO-GO}

  ...repeat the daily block for all seven dates...
  ```

- Always include the status for all three activities each day. For `CAUTION` and `NO-GO`, include a concise warning derived from that day's existing rating reason, naming the relevant observed value and active threshold; do not add warning text to `GO` ratings or invent/alter a reason.
- When a Best 3-Day Pour Window is available, copy its selected dates, Go/Caution/No-Go counts, coldest overnight low, and worst daily rain probability. If the selected result is `Best available`, include `Best available only — no No-Go-free 3-day window could be confirmed.` and do not call it safe or recommended. If no window can be ranked, use `BEST 3-DAY POUR WINDOW: Unable to rank — forecast data incomplete` and omit unavailable dates, counts, or metrics.
- Use the browser Clipboard API to copy the brief. Only after a successful copy, change the button label to `Copied!` for two seconds, then restore `Copy Crew Brief`. If copying fails or clipboard access is unavailable, do not show success; present an explicit, accessible copy-failure message and keep the forecast usable.

## Look and feel
- Page background `#FFFFFF`; card/map canvas `#F3F4F6`; body text `#111827`; muted text `#6B7280`; primary navy `#1E3A5F`; focus/link blue `#2563EB`.
- Desktop `>=1200px`: two columns, fixed `380px` left map and fluid right forecast, `24px` outer gutter and `20px` gap. At `769–1199px`: fixed `340px` left and fluid right. Mobile `<769px`: stack map above forecast, `360px` map height, `12px` gutter.
- Cards use 12 px radius and `#D1D5DB` 1 px border; inputs use `#CBD5E1` 1 px border and 4 px radius. Use 8 px spacing increments, 12 px minimum body text, 16 px minimum control text, and 2 px `#2563EB` keyboard focus rings.
- Keep status badge colors restricted to the exact Output badge hex values. Hover/focus feedback may strengthen borders or translate 3 px, with no layout shift.
- Initial state: Provo, UT in the `Location search` field, coordinates `40.2338, -111.6585`, zoom `11`, marker popup open, a visible small `Reset to Provo` button beside or below the map controls, the `Threshold Settings` panel collapsed, default thresholds loaded, and an automatic Open-Meteo request.

## Validation
- Required location fixture: Provo, UT (`40.2338° N, 111.6585° W`) is both initial state and the hand-test location.
- **Hand-calculation test:** Forecasted Day 1: minimum temperature `32°F`, maximum sustained wind `15 mph`, maximum gust `22 mph`, maximum rain probability `10%`.
  - Crane Lifts: Go (`15 < 20 mph`, `22 < 25 mph`).
  - Concrete Pours: No-Go (`32°F < 35°F`).
  - Roofing: Go (`10% < 20%`, `22 mph < 25 mph`).
- Automated tests must keep this fixture and add exact-boundary checks immediately below, exactly at, and immediately above `20 mph`, `25 mph`, `30 mph`, `35°F`, `40°F`, `60%`, `20%`, and `40%`. No threshold boundary may fall through or produce contradictory ratings.
- **National Weather Service comparison (required):** for Provo, first call `https://api.weather.gov/points/40.2338,-111.6585`, follow `properties.forecastHourly`, and compare the first seven local calendar days with the app's Open-Meteo raw table. Normalize NWS temperature and wind text to °F and mph. A match passes when temperature is within `±2°F`, sustained wind within `±3 mph`, and precipitation probability within `±10 percentage points` for matching local-hour buckets. Gust is `Not available` unless both sources provide a matching-hour gust. Log date/time, source values, normalized values, differences, pass/fail, and both URLs in `02_validation/`; investigate any out-of-tolerance result before release.
- **Best 3-Day Pour Window:** with seven complete daily summaries, confirm exactly five consecutive windows are considered; if any fully evaluable window has zero No-Go days, selection is restricted to those windows. Within the eligible set, verify more concrete Go days wins, then fewer No-Go days; for equal status counts, verify the equal-weight midrank-percentile tie-break uses each window's lowest daily minimum and highest daily rain probability, then the earlier-start final tie-break. Confirm the three chosen cards are highlighted without changing their badges; when all fully evaluable windows contain a No-Go, confirm the `Best available` message and exact reasons appear, and incomplete data is excluded or reported as unrankable.
- **Copy Crew Brief:** with a complete seven-day fixture, verify the button is in the forecast heading and copies the exact text structure with location, local dates/timezone, selected pour-window summary, all three daily activity statuses, and concise warnings only for Caution/No-Go. Verify `Best available` and unrankable-window wording, current threshold/cached forecast values, disabled states when unavailable/loading, `Copied!` for two seconds only after successful clipboard write, and an accessible failure message when clipboard write fails.
- Test at `1440px`, `1024px`, and `390px` widths. Verify marker click/drag, blank/no-match/network-failed location searches, multiple-result selection, threshold errors, card keyboard activation, detail panel, raw table, API retry/cache behavior, and NWS comparison.

## Out of scope
- User accounts, authentication, profiles, cross-device settings, or account-based sharing.
- Historical/archive weather lookup or forecasts beyond the API's seven returned local calendar days.
- Saving, sharing, exporting, or remotely storing multiple jobsites; no multi-site database or project management.
- Reverse-geocoding, routing, or more than one marker per view. City/location-name search via the specified Open-Meteo Geocoding API is in scope.
- Radar, satellite, severe-weather, tornado, hurricane, lightning, or live emergency-alert tracking.
- Real-time sensors, anemometers, worker tracking, dispatch, scheduling, cost estimation, or automated safety decisions.
- Concrete cure-state modeling, roofing surface-moisture sensing, crew safety compliance certification, or permit generation.

## Build stages
### Stage 1 — Mockup of the page layout
- **Build:** Create the static responsive shell: map area, location search controls, `Reset to Provo` button, collapsed `Threshold Settings`, and seven-card forecast layout.
- **Check:** Review at 1440 px, 1024 px, and 390 px widths. Confirm the map and forecast stack on mobile and threshold controls stay hidden until opened.

### Stage 2 — Location controls and map behavior
- **Build:** Add Leaflet, the Provo marker and popup, city/location search using Open-Meteo Geocoding, map click/drag coordination, a multiple-result chooser, and `Reset to Provo`.
- **Check:** Confirm city searches show matching choices or load a single match, missing locations show a friendly error without replacing the current forecast, map selections load by their coordinates, and Reset returns to Provo and fetches its forecast.

### Stage 3 — Forecast fetching and calculations
- **Build:** Request the specified seven-day Open-Meteo hourly fields and US customary units; implement local-day summaries, missing-data behavior, exact activity rating boundaries, API timeout/retry, and last-successful-response fallback. Log the fetched forecast data and computed daily summaries/ratings to the browser console for verification.
- **Check:** Verify the Provo hand-calculation ratings (Crane Go, Concrete No-Go, Roofing Go), values below/at/above every threshold, correct local-date grouping, incomplete-data handling, and visible console values matching the response and calculations.

### Stage 4 — Forecast cards and day details
- **Build:** Render the seven-day activity grid and status badges; add keyboard-accessible day selection, desktop detail drawer/mobile bottom sheet, exact rating reasons, and the raw hourly forecast table with copy control.
- **Check:** Confirm all three activity badges and reasons match the daily summaries; verify raw hourly values retain source precision and the detail panel/table work at desktop, tablet, and mobile widths.

### Stage 5 — Validation and polish
- **Build:** Complete the National Weather Service comparison, browser and responsive checks, API-failure checks, and keyboard/focus review; record comparison results in `02_validation/`.
- **Check:** Validate the stated NWS tolerances and investigate mismatches; retest the Provo fixture, threshold boundary cases, map reset, threshold settings, retry/cache behavior, and the 1440 px, 1024 px, and 390 px layouts.
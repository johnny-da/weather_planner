# Jobsite Weather Planner

## Purpose
A web app that helps general contractors and project managers evaluate 7-day weather forecasts to determine Go, Caution, or No-Go conditions for crane lifts, concrete pours, and roofing.

## Inputs
- Jobsite location selected via an interactive map (Leaflet) or typed coordinates.
- Crane lift wind limits (mph).
- Cold weather concreting temperature limits (°F).
- Roofing rain probability threshold (%).

## Calculations
- Fetch 7-day hourly forecast from the Open-Meteo API (temperature, wind speed, wind gusts, precipitation probability).
- Crane Lifts:
  - Go: Sustained wind < 20 mph and gusts < 25 mph.
  - Caution: Sustained wind 20–25 mph or gusts 25–30 mph.
  - No-Go: Sustained wind > 25 mph or gusts > 30 mph (OSHA / manufacturer standard).
- Concrete Pours:
  - Go: Min daily temp > 40°F.
  - Caution: Min daily temp 35°F–40°F (ACI 306R cold-weather precautions).
  - No-Go: Min daily temp < 35°F or rain probability > 60%.
- Roofing:
  - Go: Rain probability < 20% and wind gusts < 25 mph.
  - Caution: Rain probability 20%–40%.
  - No-Go: Rain probability > 40% or wet conditions.

## Outputs
- Interactive map centered on Utah/western US with a draggable marker.
- 7-day calendar card grid showing daily status badges (Green = Go, Amber = Caution, Red = No-Go) for each activity.
- Detail drawer/modal on click showing hourly breakdown and specific threshold violation reasons.

## Look and feel
- Modern, clean field-ready dashboard: light background, clear cards, prominent status badges.
- Responsive layout that stacks on mobile. Default jobsite pre-loaded (Provo, UT).

## Validation
- Location: Provo, UT (40.2338° N, 111.6585° W).
- Hand test case:
  - Forecasted Day 1: Min Temp = 32°F, Max Wind Sustained = 15 mph, Max Gust = 22 mph, Max Rain Prob = 10%.
  - Expected:
    - Crane Lifts: Go (15 < 20, 22 < 25).
    - Concrete: No-Go (32°F < 35°F, violates ACI 306R).
    - Roofing: Go (10% < 20%).

## Out of scope
- Historical weather archive lookup.
- Severe weather / tornado radar tracking.
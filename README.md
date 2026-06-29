# WeatherBen's Weather Hub

A real-time severe weather tool built for people who take storm awareness seriously. Live weather conditions, NWS severe weather alerts with polygon-accurate filtering, SPC convective outlooks, NEXRAD radar, a 7-day forecast, and a 48-hour hourly strip — all in a single self-contained HTML file with no backend, no API keys, and no build step.

**Live site:** https://weatherbenwxhub.com  
**Current version:** v1.3.1

---

## What it does

### Weather & Forecast
- **Your Backyard Forecast** — current temperature and feels-like as co-equal primary readings, wind, humidity with dew point comfort level (Comfortable → Dangerous, color-coded), UV index, visibility, and precipitation today — updated every 5 minutes
- **Heat safety context** — on days where feels-like reaches 90°F+, a heat safety line appears below the temperature with actionable guidance scaled to the severity
- **24-hour rain chances chart** — hourly precipitation probability as a bar chart with a plain-English "next rain" summary (Rain possible / expected / likely)
- **48-hour hourly strip** — scrollable, tappable hour cards with temperature, feels-like, wind, and precipitation; tap any hour to expand for full detail
- **7-day forecast** — expandable daily cards with feels-like range, precipitation totals, timing windows, wind, UV, sunrise/sunset, and context-aware cards (Extreme Heat, Warm Day Ahead, Great Day to Be Outside) replacing generic filler on non-storm days
- **Precipitation timing** — each expanded day card shows when rain is expected (e.g. "Rain expected · 10 AM – 2 PM") derived from hourly model data, with a transparent uncertainty message when model outputs conflict

### Severe Weather
- **NWS alerts** — pulled every 60 seconds, queried against both county zones (watches) and forecast zones (warnings) simultaneously so no alert type is missed
- **Polygon filtering** — each alert is tested against your exact coordinates using ray-casting point-in-polygon; if you're outside the warned polygon it's filtered out even if it covers part of your county
- **IBW tier classification** — Tornado Emergency → PDS Tornado Warning → Observed → Radar Indicated, read from structured WarnGen parameters with free-text fallback
- **SVS deduplication** — fetches both `message_type=alert` and `message_type=update`, deduplicates by reference chain so SVS products with revised polygons and PDS upgrades are always current
- **Tap for full report** — tap any alert to read the complete NWS product text including affected area, timing, and source
- **SPC Convective Outlook** — categorical risk (Marginal → High) for Days 1–3, probabilistic tornado/hail/wind percentages for Days 1–2, pulled from NOAA's MapServer with max-dn polygon logic so nested risk areas always show the correct tier
- **Recent Storm Reports** — confirmed tornado, hail, and damaging wind reports from NWS Local Storm Reports appear automatically within 75 miles of your location

### Navigation & Personalization
- **Navigation menu** — hamburger menu consolidates My Locations, search, themes, guide, and feedback; search bar collapses after a location loads; alert badge on the menu icon when active alerts are present
- **Header star** — tap ☆ next to the location name to save it instantly without navigating away
- **My Locations** — save up to 3 locations, view current conditions (temp, high/low, rain %, wind, active alerts) for all of them at a glance, tap any card to switch
- **Your Day at a Glance** — collapsible daily briefing with today/tomorrow forecast, next rain timing, SPC status, and a clothing suggestion — relevant any time of day, not just mornings
- **6 themes** — Midnight Navy (default), Twilight Purple, Autumn Harvest, Storm Slate, Radar Dark, Desert Dusk; choice persists across the app including guide and feedback pages
- **Date and time** — live clock in the user's local timezone displayed in the header

---

## Architecture

Single HTML file. No framework, no build step, no backend.

```
index.html        — the entire application (~200 KB)
guide.html        — how-to-use page
feedback.html     — Netlify Forms feedback form
success.html      — post-submission thank-you page
weatherhub-roadmap.md — full feature planning document
```

Open `index.html` in any browser to run it locally. Deploy by pushing to any static host — the live site uses Netlify with automatic deploys on push to `main`.

---

## Data sources

| Source | What it provides | Refresh rate |
|--------|-----------------|--------------|
| [Open-Meteo](https://open-meteo.com) | Current conditions, hourly forecast, 7-day daily forecast, precipitation history | Every 5 min |
| [NWS API](https://www.weather.gov/documentation/services-web-api) | Severe weather alerts, radar station, forecast office | Alerts: 60 sec |
| [SPC / NOAA MapServer](https://mapservices.weather.noaa.gov/vector/rest/services/outlooks/SPC_wx_outlks/MapServer) | Convective outlook categorical risk, probabilistic tornado/hail/wind | On location load |
| [NWS LSR MapServer](https://mapservices.weather.noaa.gov/vector/rest/services/obs/nws_local_storm_reports/MapServer) | Local storm reports (tornado, hail, wind damage) within 75 miles | Every 30 min |
| [Open-Meteo Geocoding](https://open-meteo.com/en/docs/geocoding-api) | City search and name resolution | On search |
| [BigDataCloud](https://www.bigdatacloud.com) | Reverse geocoding for GPS location | On GPS use |

No API keys required. All data sources are free and public.

---

## How alerts work

**Dual zone query:** The app queries alerts against both the county zone (`KYC227`) and the forecast zone (`KYZ070`) simultaneously. NWS issues watches against county zones and warnings against forecast zones — querying only one type silently misses the other.

**Polygon filtering:** Each geometry-bearing alert is tested against the user's exact coordinates using a ray-casting point-in-polygon algorithm. Alerts that cover part of the county but not the user's specific location are filtered out.

**SVS deduplication:** Fetches `message_type=alert,update` and deduplicates by reference chain, always keeping the most recent version. SVS continuation products carrying PDS designations, confirmed tornado source, and revised polygons come in as `update` type — fetching only the original issuance misses these.

**IBW tier classification:** Reads `properties.parameters.tornadoDamageThreat` and `properties.tornadoDetection` — structured WarnGen fields set at issuance — with free-text scanning of `description` and `headline` as a fallback.

**SPC polygon max-dn:** The SPC outlook query can return multiple overlapping polygons for a point (e.g. both a TSTM and a SLGT polygon). The app takes the maximum `dn` value across all returned features so the correct tier is always shown.

---

## Storm code accuracy

Open-Meteo's daily and hourly WMO weather codes can assign thunderstorm codes (95/96/99) to days or hours where rain probability is near zero — the code and probability fields come from different model outputs that don't always agree. The app applies a consistent guard throughout:

- **Daily codes:** requires ≥30% rain probability AND non-zero expected precipitation before showing storm messaging, icons, or hail advisories
- **Hourly codes:** same guard applied to the 48-hour strip
- **Current conditions:** requires non-zero current-hour precipitation
- **All three locations:** briefing, 7-day cards, and My Locations panel all apply the guard

When a storm code fires but the guard fails, a more accurate description is substituted based on rain probability (Mostly Sunny / Isolated Storm Chance).

---

## Known limitations

- **60-second alert polling gap** — no push/websocket mechanism. A warning issued immediately after a poll cycle won't appear for up to 60 seconds.
- **US-only** — NWS coverage only. Alaska and Hawaii are supported by the NWS API but not tested.
- **SPC probabilistic data: Days 1–2 only** — tornado/hail/wind percentages aren't published for Day 3. Categorical risk is available for Days 1–3.
- **Model data conflicts** — Open-Meteo's hourly probability and daily precipitation sum come from different model outputs and can disagree. When they conflict and an SPC outlook exists, the app surfaces a transparent uncertainty message rather than a false "no precipitation expected."
- **Radar embed** — the embedded NEXRAD view is a static iframe with a 2–4 minute data delay. Use NWS directly for rapidly evolving situations.
- **Safari geolocation** — Safari caches location permission denials. Re-enabling requires going to Settings → Privacy → Location Services → Safari and reloading the page. Installing as a PWA (Add to Home Screen) avoids this entirely.

---

## Versioning

Semantic versioning: `major.minor.patch`

- **Patch** (x.x.1) — bug fixes
- **Minor** (x.1.0) — feature releases
- **Major** (2.0.0) — significant architectural changes

### Release history

| Version | Highlights |
|---------|-----------|
| v1.1.0 | My Locations, Hourly Detail, SPC Hazard Probabilities, Clickable Alerts |
| v1.2.0 | Precipitation Timing, Rain Chart, Renamed Sections, Feels-Like Prominence, Storm Reports |
| v1.2.1 | Alert zone fixes, SPC accuracy fixes, Safari geolocation fix |
| v1.2.2 | Mobile fonts, feedback form, thunderstorm mislabeling fix, data loading fix |
| v1.3.0 | Navigation Menu, Themes (6 options), Humidity & Heat Prominence |
| v1.3.1 | Storm code guard across all display surfaces, theme persistence to guide/feedback pages, context-aware day cards, header star for saving locations, mobile header layout fix |

---

## Planned features

### v1.4.0
- Expand saved locations from 3 to 6 with responsive My Locations panel layout (single column on phone/tablet portrait, two-column grid on tablet landscape/desktop)
- Glossary page — plain-English explanations of weather terminology (Watch vs. Warning, PDS, SPC risk levels, EF Scale, IBW tags, dew point comfort, precipitation probability) with contextual info links on the main page
- SPC storm reports guide and changelog entry (feature already built and live)

### v1.5.0 (tentative)
- Winter weather precipitation type breakdown — rain/snow/sleet proportions on days with frozen or mixed precipitation WMO codes

### PWA (before v2.0.0)
- Web app manifest, service worker, home screen icon
- Offline caching for last-loaded location
- Fixes Safari geolocation behavior for iOS users who install to home screen

### v2.0.0 — Interactive Radar
- Leaflet.js map with IEM NEXRAD tile layer
- Animation loop through last 6–12 radar frames
- NWS warning polygon overlay (reuses existing alert data)
- LSR storm report markers on the map
- Full-screen mode

### Future
- Push notifications (requires backend — Supabase + web-push)
- Google Play distribution via Trusted Web Activity (TWA)
- Native iOS app evaluation

See [`weatherhub-roadmap.md`](weatherhub-roadmap.md) for full planning detail.

---

## Disclaimer

WeatherBen's Weather Hub is a personal project. It is not an official source of weather information and is not affiliated with, endorsed by, or operated by the National Weather Service, NOAA, the Storm Prediction Center, or any government agency, news organization, or official weather provider.

Do not use this tool as your sole source of information during severe weather events. Always consult official NWS products and emergency management authorities.

Official severe weather information is always available at [weather.gov](https://www.weather.gov).

---

## Running locally

```bash
# No build step — just open the file
open index.html

# Or serve it with any static server
npx serve .
python3 -m http.server 8080
```

---

*Built with care for severe weather awareness. Stay safe out there.*

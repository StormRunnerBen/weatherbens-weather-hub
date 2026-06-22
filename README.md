# WeatherBen's Weather Hub

A real-time severe weather tool built for people who take storm awareness seriously. Live weather conditions, NWS severe weather alerts with polygon-accurate filtering, SPC convective outlooks, NEXRAD radar, and a 7-day forecast — all in a single self-contained HTML file with no backend, no API keys, and no build step.

**Live site:** https://weatherbenwxhub.com

---

## What it does

- **Current conditions** — temperature, feels-like, humidity, dew point, wind, UV index, visibility, updated every 5 minutes
- **Severe weather alerts** — pulled from the NWS API every 60 seconds, filtered to your exact coordinates using polygon point-in-polygon testing so you only see warnings that actually cover your location
  - Full IBW (Impact-Based Warning) tier classification: Tornado Emergency → PDS → Observed → Radar Indicated
  - Severity-sorted when multiple alerts are active
  - Tap any alert to read the full NWS report text
- **SPC Convective Outlook** — categorical risk level (Marginal → High) plus probabilistic tornado, hail, and wind percentages for Days 1–2, pulled from NOAA's MapServer
- **7-day forecast** — expandable daily cards with feels-like range, precipitation totals, wind, UV, sunrise/sunset, and SPC hazard breakdown on storm days
- **48-hour hourly strip** — scrollable, tappable hour cards with wind, feels-like, and precipitation detail
- **Live NEXRAD radar** — embedded from NWS, matched to your location's forecast office
- **Morning Briefing** — collapsible daily summary with today/tomorrow forecast, active alert status, and a clothing suggestion
- **User feedback** — Netlify Forms integration for bug reports and feature requests

---

## Architecture

Single HTML file. No framework, no build step, no backend.

```
index.html        — the entire application
guide.html        — how-to-use page
feedback.html     — Netlify Forms feedback form
success.html      — post-submission thank-you page
```

Open `index.html` in any browser to run it locally. Deploy by pushing to any static host — the live site uses Netlify with automatic deploys on push to `main`.

---

## Data sources

| Source | What it provides | Refresh rate |
|--------|-----------------|--------------|
| [Open-Meteo](https://open-meteo.com) | Current conditions, hourly forecast, 7-day daily forecast, precipitation history | Every 5 min |
| [NWS API](https://www.weather.gov/documentation/services-web-api) | Severe weather alerts, radar station, forecast office | Alerts: 60 sec |
| [SPC / NOAA MapServer](https://mapservices.weather.noaa.gov/vector/rest/services/outlooks/SPC_wx_outlks/MapServer) | Convective outlook categorical risk, probabilistic tornado/hail/wind | On location load |
| [BigDataCloud](https://www.bigdatacloud.com) | Reverse geocoding for GPS location only | On GPS use |

No API keys required. All data sources are free and public.

---

## How alerts work

The NWS alert pipeline has a few non-obvious layers worth understanding:

**Zone + polygon:** The app queries alerts by county zone (not raw lat/lon) because zone queries catch small-polygon products — Tornado Warnings, Severe Thunderstorm Warnings, Flash Flood Warnings — that a plain point query can silently miss. After fetching, each polygon-based alert is tested against the user's exact coordinates using a ray-casting point-in-polygon algorithm. If you're outside the warned polygon, the alert is filtered out, even if it covers part of your county.

**SVS updates:** The app fetches both `message_type=alert` and `message_type=update`, then deduplicates by reference chain keeping the most recent version. This is critical — SVS continuation products (which carry PDS designations, confirmed tornado source, and revised polygons) come in as `update` type. Fetching only the original issuance misses these.

**IBW tier classification:** Tiers are read from `properties.parameters.tornadoDamageThreat` and `properties.tornadoDetection` — structured fields set by WarnGen at issuance — with free-text scanning of `description` and `headline` as a fallback for older products.

---

## Known limitations

- **60-second alert polling gap** — there's no push/websocket mechanism. A warning issued immediately after a poll cycle won't appear for up to 60 seconds.
- **US-only** — NWS coverage only. Alaska and Hawaii are supported by the NWS API but not tested.
- **SPC data: Days 1–2 only** — probabilistic tornado/hail/wind outlooks aren't published for Day 3. Categorical risk is available for Days 1–3.
- **Open-Meteo model data** — precipitation totals and storm labels in the 7-day forecast are global model estimates, not official NWS forecasts. The SPC outlook is authoritative for severe weather.
- **Radar embed** — the embedded NEXRAD view has a built-in 2–4 minute data delay. Use the NWS full radar page for rapidly evolving situations.

---

## Planned features

- Hourly forecast detail (in progress)
- Saved locations / multi-location dashboard
- Recent SPC storm reports
- Winter weather precipitation type breakdown
- Progressive Web App (PWA) — installable, home screen icon, offline caching
- Push notifications (requires backend — future phase)

See [`weatherhub-roadmap.md`](weatherhub-roadmap.md) for full planning detail.

---

## Beta disclaimer

WeatherBen's Weather Hub is a personal project currently in beta testing. It is not an official source of weather information and is not affiliated with, endorsed by, or operated by the National Weather Service, NOAA, the Storm Prediction Center, or any government agency, news organization, or official weather provider.

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

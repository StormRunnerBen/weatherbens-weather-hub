# WeatherBen's Weather Hub — Development Roadmap

*Last updated: June 2026*

---

## Where We Are Now

The current tool is a fully functional, single-file weather app deployed on Netlify. It covers:

- Real-time current conditions and 7-day forecast (Open-Meteo)
- NWS severe weather alerts with polygon-accurate filtering, IBW tier classification, and full NWS report detail on tap
- SPC Convective Outlook integration for Days 1–3 on storm days
- Live NEXRAD radar via NWS
- Your Day at a Glance — collapsible daily summary with today/tomorrow forecast, rain timing, active alert status, and a clothing suggestion
- Clickable 7-day forecast cards with expanded detail, including hail context
- Severity-matched alert notification banner
- Beta disclaimer, guide page, and Netlify Forms feedback form
- Mobile-responsive layout

No backend, no database, no API keys — everything runs client-side from a single HTML file.

---

## Planned Features (Near Term)

These five features were identified as the highest-value additions that don't require a backend, use data already available in the app, and address the most common user needs.

---

### 1. Hourly Forecast View

**Priority: High — biggest immediate user value**

A scrollable hour-by-hour breakdown for the next 24–48 hours showing temperature, precipitation probability, wind speed/direction, and conditions. Open-Meteo already returns all of this data in the existing fetch — it's a display-only addition with no new API calls required.

**Why it matters:** The 7-day cards show daily highs/lows but give no timing information. "Will it rain during my commute?" and "When does the storm arrive?" are the most common questions a weather tool gets asked that the current app can't answer.

**Data available:** `hourly.temperature_2m`, `hourly.weather_code`, `hourly.precipitation_probability`, `hourly.precipitation`, `hourly.wind_speed_10m`, `hourly.wind_direction_10m` — all already fetched.

**Implementation approach:** A horizontally scrollable strip beneath each expanded day card, or a dedicated hourly section between the current conditions and 7-day list. Tapping a day card could expand to show that day's hourly breakdown.

**Estimated effort:** 1–2 days.

---

### 2. Saved Locations / Multi-Location Dashboard

**Priority: High — lays groundwork for push notifications later**

True saved locations where users can pin multiple cities (a family member's town, a vacation destination, a work commute point) and see their current conditions at a glance without re-searching each time.

**Why it matters:** Favorites currently act as quick-search shortcuts, but users can only view one location at a time. A multi-location dashboard is how most people actually use weather tools — checking on family in a storm-affected area being the most immediate use case given this app's severe weather focus.

**Implementation approach:** A pinned locations row below the search bar showing current temp and conditions for each saved city, pulling from Open-Meteo. Tap to load that location fully. localStorage handles persistence with no backend needed.

**Future connection:** The saved locations list is also the natural starting point for push notification subscriptions — when that backend is built, each saved location becomes a location the user wants alerts for.

**Estimated effort:** 2–3 days.

---

### 3. Feels-Like and Humidity Prominence

**Priority: Medium — high user impact, low effort**

Elevate the feels-like temperature as a primary display element alongside the actual temperature, rather than treating it as one of eight equal items in the conditions grid. Humidity and dew point could also be more visually prominent given how much they affect comfort, especially in summer.

**Why it matters:** User feedback on weather apps consistently centers on "what does it actually feel like outside" as the most-used data point. The current app has this information but buries it. A small layout change would make it substantially more useful day-to-day without adding any complexity.

**Data available:** Already fetched — `current.apparent_temperature`, `current.relative_humidity_2m`, `current.dew_point_2m`.

**Implementation approach:** Show feels-like directly beneath or alongside the main temperature in the hero card. Could take the form of `72°F · Feels like 78°` as a subtitle rather than a grid cell.

**Estimated effort:** Half a day.

---

### 4. Recent Storm Reports (SPC)

**Priority: Medium — differentiating feature**

After an active severe weather day, show nearby confirmed tornado, wind, and hail reports from the Storm Prediction Center's storm reports feed. This gives users context on what actually happened in their area after a storm event — something most consumer weather apps don't surface well.

**Why it matters:** During and after events like the June 2026 Indiana tornado outbreak, users want to understand what occurred — was there a confirmed tornado? What was the path? The SPC publishes this data publicly and it's the same source meteorologists and storm spotters use.

**Data source:** `https://www.spc.noaa.gov/climo/reports/` — publishes today's and recent days' tornado, wind, and hail reports as CSV/GeoJSON. No API key required. CORS policy will need verification before building.

**Implementation approach:** A "Recent Storm Reports" section (below the alerts section, only visible when reports exist near the searched location) showing confirmed tornadoes, significant wind, and large hail events within a radius of the user's coordinates for the past 24–48 hours.

**Estimated effort:** 2–3 days, with the main uncertainty being the SPC data format and CORS verification.

---

### 5. Winter Weather Precipitation Type Breakdown

**Priority: Medium — high seasonal value for Midwest user base**

A winter weather mode that distinguishes between snow, sleet, freezing rain, and mixed precipitation rather than labeling everything as "Light snow" or "Snow." Flags mixed precip risk and explains timing of transitions (e.g. "Rain changing to freezing rain after 6 PM").

**Why it matters:** Winter weather is disproportionately dangerous relative to how much attention it gets in the app. A freezing rain event looks identical to a snow event in the current display. For users in Kentucky, Indiana, and Tennessee (where this app is being beta tested), ice storms are a significant recurring hazard.

**Data available:** Open-Meteo already returns `rain_sum`, `showers_sum`, and `snowfall_sum` separately in the daily fetch. The hourly `freezing_level_height` variable (not currently fetched) would enable mixed precip detection.

**Implementation approach:** On days where WMO codes suggest frozen or mixed precipitation, the expanded day card shows a precipitation type breakdown with rain/snow/sleet proportions. A "Winter Weather" flag would appear on cards where freezing rain or mixed conditions are modeled.

**Estimated effort:** 2–3 days, including adding `freezing_level_height` to the hourly fetch.

---

## Next Step: Progressive Web App (PWA)

### What it gets you

- Users can tap "Add to Home Screen" on iOS and Android
- App launches full-screen with no browser chrome — looks and behaves like a native app
- App icon on the home screen
- Potential for basic offline caching (last-loaded weather still visible if connection drops briefly)
- Foundation layer required before push notifications can be added later

### What it does NOT get you (yet)

- App Store or Play Store listing
- Push notifications (that's the next phase)
- Any server-side functionality

### Files to create

**`manifest.json`** — tells the browser how to treat the app when installed

```json
{
  "name": "WeatherBen's Weather Hub",
  "short_name": "WeatherHub",
  "start_url": "/",
  "display": "standalone",
  "background_color": "#0a1628",
  "theme_color": "#0a1628",
  "icons": [
    { "src": "/icons/icon-192.png", "sizes": "192x192", "type": "image/png" },
    { "src": "/icons/icon-512.png", "sizes": "512x512", "type": "image/png" },
    { "src": "/icons/icon-512-maskable.png", "sizes": "512x512", "type": "image/png", "purpose": "maskable" }
  ]
}
```

**`sw.js`** (service worker) — enables installability and offline caching. Lives at the root of the repo alongside `index.html`.

**Icon assets** — minimum set needed:
- 192×192 PNG (Android home screen)
- 512×512 PNG (splash screen)
- 512×512 maskable PNG (Android adaptive icon — safe zone matters here)
- 180×180 PNG (iOS "apple-touch-icon")

### Changes to `index.html`

Two additions in the `<head>`:

```html
<link rel="manifest" href="/manifest.json">
<link rel="apple-touch-icon" href="/icons/icon-180.png">
<meta name="theme-color" content="#0a1628">
```

One addition before `</body>`:

```html
<script>
  if ('serviceWorker' in navigator) {
    navigator.serviceWorker.register('/sw.js');
  }
</script>
```

### Estimated effort

One focused weekend. The service worker can start simple (cache-first for the HTML/CSS/JS assets, network-first for API calls) and be refined later.

### Validation

Use Chrome DevTools → Application → Manifest to verify the PWA is installable. Lighthouse will flag any missing requirements.

---

## Future Phase: Push Notifications

Push notifications are a separate, backend-dependent feature that builds on top of the PWA foundation. They are not required for the PWA and can be added later independently.

### Why it's non-trivial

The current app is entirely client-side — it fetches data when a user has it open. Push notifications require something running when the user isn't looking at the app. That means a backend.

### The four pieces needed

**1. User location subscription storage**

A database storing each device's push token paired with one or more locations they want alerts for. This introduces the first identity requirement — currently the app is fully anonymous. A lightweight device ID or account system would be needed.

**2. Server-side alert polling job**

A scheduled job (matching the current 60-second poll interval) that checks NWS on behalf of every subscriber. Needs to be smart about batching — users in the same county zone can share a single NWS API call rather than each triggering their own. NWS has rate limits.

**3. Push delivery mechanism**

Differs by platform:

| Platform | Mechanism | Cost | Notes |
|----------|-----------|------|-------|
| Web/PWA (Android Chrome) | Web Push (VAPID) | Free | Works well, well-supported |
| Web/PWA (iOS Safari) | Web Push (VAPID) | Free | Supported since iOS 16.4, some quirks |
| Native Android | Firebase Cloud Messaging (FCM) | Free | Most reliable Android path |
| Native iOS | Apple Push Notification service (APNs) | $99/yr dev account | Requires App Store app |

For a PWA-first approach, VAPID/Web Push covers both platforms reasonably well.

**4. Alert deduplication and escalation logic**

The hardest product problem:
- Don't re-alert for the same warning already seen
- Do alert again if severity escalates (Radar Indicated → PDS → Tornado Emergency)
- Don't spam watch issuances when a warning is already active
- Consider quiet hours (debatable for severe weather)

The tier classification logic already in the app can be ported server-side with modest effort.

### Proposed stack (lightweight, low/no cost at beta scale)

| Component | Tool | Why |
|-----------|------|-----|
| Database | Supabase (free tier) | Handles tokens + locations, built-in edge functions |
| Polling job | Supabase Edge Functions or Cloudflare Workers | Runs on schedule, no always-on server needed |
| Push delivery | `web-push` npm library | Handles VAPID key management and push dispatch |
| Frontend | Existing Netlify site | Add service worker push handler |

Monthly cost at beta scale: ~$0 on free tiers.

### What can be reused from the existing frontend

- All NWS alert fetch and parsing logic
- Polygon point-in-polygon filtering
- IBW tier classification (tornadoDamageThreat / tornadoDetection parameter reads)
- Alert severity sort order

These can be copied almost verbatim and run server-side in Node.js.

### Estimated effort

2–4 weeks of focused work for someone comfortable with backend development. The polling job and deduplication state management are the bulk of the work. Push delivery itself is well-documented once the infrastructure is in place.

---

## v2.0.0 — Interactive Radar

The headline feature of the v2.0.0 release. Replaces the current NWS iframe embed with a fully interactive, pannable, zoomable radar map built on **Leaflet.js** with NEXRAD tiles from the **Iowa Environmental Mesonet (IEM)** and/or **RainViewer**.

### Why v2.0.0

Interactive radar is a significant enough architectural addition to warrant a major version. It introduces a third-party map library (Leaflet), a new data source (IEM/RainViewer radar tiles), and a fundamentally different UI paradigm. Everything before this is data display — this is the first true interactive feature.

### Architecture

Maintains the single-file constraint — Leaflet loads from CDN, radar tiles and map base tiles load on demand. No backend required. No API key required (IEM is free and open).

**Stack:**
- **Leaflet.js** (CDN) — map renderer, zoom/pan, touch handling
- **IEM NEXRAD tiles** — `mesonet.agron.iastate.edu` composite radar, free, no key
- **RainViewer API** (optional) — smoother animation, 2hr history, free tier
- **Existing NWS alert data** — warning polygons rendered directly on the map
- **Existing LSR storm report data** — markers on the map

### Planned features

**Phase 1 — Core radar (MVP)**
- Leaflet map centered on user's location
- IEM NEXRAD composite radar tile layer
- Zoom and pan with touch support on mobile
- Auto-centers on current location

**Phase 2 — Animation**
- Loop through last 6–12 radar frames at ~500ms intervals
- Play/pause controls
- Frame timestamp display

**Phase 3 — Overlays**
- NWS warning polygons (reuse existing alert fetch — just render on map)
- County/state boundary lines
- LSR storm report markers (tornado, hail, wind damage)

**Phase 4 — Polish**
- Loading states so animation doesn't start until tiles are ready
- Radar legend (dBZ color scale)
- Mobile-optimized controls
- Full-screen mode (especially useful post-PWA)

### Prerequisites

- **PWA should be built first** — full-screen interactive radar without browser chrome is dramatically better on mobile. The PWA makes the radar feel native.

### Effort estimate

| Phase | Effort |
|-------|--------|
| Core map + radar tiles | 1–2 days |
| Animation loop | 1 day |
| Warning polygon overlay | Half day |
| LSR markers | Half day |
| Polish + mobile | 1–2 days |
| **Total** | **~2 focused weekends** |

### Data sources

| Source | What | Cost |
|--------|------|------|
| IEM NEXRAD tiles | Radar composites | Free, no key |
| RainViewer | Animated radar frames | Free tier available |
| Leaflet.js | Map renderer | Free, open source |
| NWS API | Warning polygons | Free (already in use) |
| NOAA LSR MapServer | Storm report markers | Free (already in use) |

Not required for a beta, but good to understand the landscape.

### Google Play — accessible

A Trusted Web Activity (TWA) wraps your existing website in a thin Android shell. Tools like **Bubblewrap** (by Google) automate most of this. One-time $25 Play Store fee. Review is usually quick.

Requirements: your site must pass PWA installability checks (so do the PWA step first), and it must be served over HTTPS (Netlify handles this).

### Apple App Store — significantly more involved

Apple scrutinizes "wrapper" apps and has rejected apps that are perceived as "just a website." Weather apps with clear utility tend to fare better than generic web wrappers.

Requirements:
- Mac with Xcode
- Apple Developer Program membership ($99/year)
- Either a proper native wrapper (Capacitor, React Native) or a compelling UIKit/SwiftUI shell
- App Store review (typically 1–3 days, can be longer)

### Recommendation

PWA → Play Store TWA is a natural progression and relatively low-friction. App Store is a separate project with meaningful overhead — worth evaluating once the user base justifies it.

---

## Summary: Recommended Order of Operations

1. **Done:** Netlify Forms feedback form — live and collecting submissions
2. **Done:** Hourly forecast detail — tappable hour cards with feels-like, wind, and precipitation detail
3. **Done:** Saved locations / My Locations panel — up to 3 locations with live conditions and alert badges
4. **Done:** Precipitation timing — 7-day cards show when rain is expected; 24-hour rain chances chart in Your Backyard Forecast
5. **Done:** Feels-Like and Humidity Prominence — feels-like co-equal with actual temp; humidity card with dew point comfort level and heat safety context
6. **Done — v1.3.0:** Navigation menu, Themes (6 options), Humidity & Heat Prominence
7. **Done — v1.3.1 (current):** Hourly storm icon guard, Day at a Glance storm guard, theme persistence to guide/feedback pages, context-aware day cards replacing SPC No Severe Risk, header star for saving locations
8. **Target for v1.4.0:**
   - **Expand saved locations from 3 to 6** — `MAX_FAVORITES` constant change plus responsive My Locations panel layout (see note below)
   - Recent SPC storm reports — already built, needs guide/changelog entry
   - Winter weather precipitation breakdown — high seasonal value for Midwest users
9. **Near term after v1.4.0:** Build the PWA (manifest + service worker + icons) — one weekend
10. **If growth warrants it:** Add Play Store distribution via TWA
11. **If growth warrants it:** Build the push notification backend
12. **Long term:** Evaluate native iOS app if the user base reaches a scale that justifies the overhead

### Note: My Locations panel layout for 6 locations

Expanding to 6 saved locations requires a responsive panel layout update to avoid the panel feeling too long on larger screens. Proposed breakpoints:

| Screen | Panel width | Card layout |
|--------|------------|-------------|
| Phone (portrait + landscape) | `min(300px, 85vw)` | Single column — always |
| iPad portrait (~768px) | `min(380px, 85vw)` | Single column — panel too narrow for two |
| iPad landscape / desktop (≥1024px) | `min(520px, 50vw)` | Two-column card grid |

The layout work should be done as part of the same v1.4.0 item — the panel should be solid at any card count before the maximum is increased. The functional change (`MAX_FAVORITES = 6`) is a one-liner; the layout update is the bulk of the work.

---

*This document is a planning reference and will need updating as the project evolves. Data sources, API behavior, and platform policies can change.*

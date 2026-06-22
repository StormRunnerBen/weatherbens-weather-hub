# WeatherBen's Weather Hub — Development Roadmap

*Last updated: June 2026*

---

## Where We Are Now

The current tool is a fully functional, single-file weather app deployed on Netlify. It covers:

- Real-time current conditions and 7-day forecast (Open-Meteo)
- NWS severe weather alerts with polygon-accurate filtering, IBW tier classification, and full NWS report detail on tap
- SPC Convective Outlook integration for Days 1–3 on storm days
- Live NEXRAD radar via NWS
- Morning Briefing summary card
- Clickable 7-day forecast cards with expanded detail
- Beta disclaimer and guide page
- Severity-matched alert banner

No backend, no database, no API keys — everything runs client-side from a single HTML file.

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

## App Store Distribution (Future Consideration)

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

## User Feedback Integration

A Google Form embedded in or linked from the app gives users a direct channel for bug reports and feature requests.

### Options

**Option A — Linked button** (lowest friction, already implemented)
A visible "Send Feedback" button in the app opens the Google Form in a new tab. Simple, no maintenance.

**Option B — Embedded modal**
The form loads in an overlay within the app using a `<iframe>`. Keeps users on the page but Google Forms iframes can behave inconsistently on mobile.

**Option C — Netlify Forms** (future consideration)
If you ever move away from Google Forms, Netlify has built-in form handling — responses go to your Netlify dashboard with no backend needed. Relevant if you want more control over the data.

### Suggested form questions

- What device/browser are you using?
- What were you trying to do when you ran into an issue?
- How would you rate the accuracy of the severe weather alerts?
- What feature would you most like to see added?
- Any other feedback?

---

## Summary: Recommended Order of Operations

1. **Now (or next session):** Add Google Form feedback link to the app
2. **Near term:** Build the PWA (manifest + service worker + icons) — one weekend
3. **If growth warrants it:** Add Play Store distribution via TWA
4. **If growth warrants it:** Build the push notification backend
5. **Long term:** Evaluate native iOS app if the user base reaches a scale that justifies the overhead

---

*This document is a planning reference and will need updating as the project evolves. Data sources, API behavior, and platform policies can change.*

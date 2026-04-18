# Handoff notes

For a coding agent (Claude Code, Cursor, or equivalent) picking this project up.

## Context

Owner: Michael, embedded software / CS background, The Hague, NL. Already has a related project: a local-first PWA energy tracker (session timer, slider, timeline visualization) — so PWA patterns and the single-file-with-iterative-refinement workflow are familiar territory.

This project was prototyped in a conversation: static globe first, then added all-Kp-zone color gradient, then layered live NOAA data. The current `index.html` is the live-data version.

## Current state (working)

- Orthographic projection globe, drag-to-rotate, 3 preset camera views (polar / Europe / N America)
- Static Kp threshold bands, 10 colored rings, centered on magnetic pole
- Live OVATION aurora nowcast overlay on a `<canvas>` layer above the SVG
- Kp readout (big number) + 72-hour forecast strip
- DSCOVR solar wind panel (speed, Bz, density) with aurora-friendly coloring
- 14 cities with precomputed magnetic latitude + required Kp threshold
- "Tonight?" column in city table — computed live from current Kp
- Auto-reload every 5 minutes (crude but works)
- Graceful degradation: if NOAA fetches fail (typical file:// CORS), static model still renders and a hint about `python3 -m http.server` is shown

## Architecture

**Frontend-only.** Zero server needed at runtime. Any static host works.

**CDN dependencies:**
- D3 7.8.5 from cdnjs
- TopoJSON 3.0.2 from cdnjs
- Country borders (Natural Earth 110m) from jsdelivr

**NOAA SWPC data feeds** (all JSON, CORS-enabled, no key):
- `/products/noaa-planetary-k-index.json` — observed Kp, 3-hour cadence
- `/products/noaa-planetary-k-index-forecast.json` — observed + predicted, 3-hour cadence, ~72h ahead
- `/json/ovation_aurora_latest.json` — ~900 KB, 1°×1° aurora probability grid, ~5 min cadence
- `/products/solar-wind/mag-1-day.json` — magnetometer (need Bz_gsm)
- `/products/solar-wind/plasma-1-day.json` — speed, density, temperature

**Rendering layers** (bottom to top):
1. Background sphere + graticule (SVG)
2. Threshold zones — 10 `d3.geoCircle` around magnetic pole (SVG)
3. Country borders (SVG)
4. OVATION aurora probability — canvas dots, one per grid cell with probability ≥ 3
5. Highlight ring for selected city (SVG, top layer)
6. Magnetic pole marker (SVG, top layer)
7. City markers + labels (SVG, top layer, interactive)

Two SVGs are stacked with a `<canvas>` sandwiched between them — bottom SVG holds bands + countries, canvas holds the OVATION overlay, top SVG holds the interactive city/pole/highlight layer. This lets the city markers stay crisp and interactive while the OVATION overlay gets canvas-rendering performance.

**Magnetic latitude:** computed via great-circle distance from a hardcoded pole at (80.7°N, 72.7°W). Not time-varying in current code.

**Threshold formula:** `min Kp = (66 − mag_lat) / 2`, clamped 0–9. Source: AuroraMe model, validated against naked-eye visibility reports.

## Known issues / caveats

1. **file:// CORS:** Chrome/Safari block cross-origin fetch from `file://` regardless of target CORS headers. Firefox is sometimes lenient. Always serve via http.
2. **Magnetic pole is static.** IGRF pole drifts ~50 km/year. Should be computed from IGRF-14 coefficients or updated annually.
3. **Circular oval is an approximation.** Real auroral oval is compressed toward nightside, asymmetric, and expands during substorms. Current model is best-effort threshold.
4. **Extreme storms underestimated.** Gannon storm (May 2024, G5) produced aurora down to Mexico; formula predicts only to mag lat 48°. Red SAR arcs aren't in this model.
5. **OVATION redraws every drag frame.** ~10k points at typical activity. Fine on modern hardware, noticeable stutter on low-end mobile. Could throttle via rAF or switch to `d3.contour` polygons.
6. **Auto-reload is `location.reload()`.** Loses drag state, causes visual flash. Should be replaced with incremental re-fetch + selective redraw.
7. **No service worker / offline support.**
8. **No error surface for partial failures.** If OVATION fails but Kp succeeds, user sees Kp but no obvious indication OVATION is missing.
9. **Canvas DPI.** Currently drawn at 1200×1200 on a 600×600 viewport (2× DPR assumption). Better would be `window.devicePixelRatio`.
10. **Cloud cover not integrated.** Biggest real-world blocker for aurora viewing and the most obvious missing data layer.

## Reasonable next steps

Rough priority — pick based on what the owner asks for:

### 1. PWA conversion
Aligns with the owner's parallel PWA project. Add `manifest.json`, service worker that caches app shell + topology, and IndexedDB for the last successful NOAA responses so the page works offline with stale data. Add "last updated X min ago" prominently so stale vs fresh is obvious.

### 2. Geolocation + personal threshold
Browser `navigator.geolocation` once, cache to localStorage. Replace/augment the city list with "you are here" marker. Compute personal magnetic latitude and minimum Kp. Primary readout becomes: "Tonight's Kp is 3, your threshold is 6, so: no." Much more useful than a generic globe.

### 3. Cloud cover layer
Integrate Open-Meteo (free, no key) for cloud cover forecast. Overlay as a second canvas or as a subtle gray-scale wash. Key insight: if Kp is 6 and your local cloud cover is 100%, it doesn't matter. Show both together.

### 4. Aurora alerts
Web Push API: subscribe, run a background check (via the forecast endpoint + a scheduled task / cloud function), notify when predicted Kp exceeds user's personal threshold within the next N hours. This is AuroraMe's core product.

### 5. Smooth OVATION rendering
Replace canvas dot-stamping with `d3.contour` over the probability grid → smooth filled isolines. Better aesthetics and performance. Colors can be a continuous green ramp by probability.

### 6. Historical playback
The NOAA Kp JSON goes back years; `ovation_aurora_latest.json` is real-time only but historical grids exist on NCEI. Add a time slider to scrub through past events. Demo: May 10, 2024 — the Gannon storm.

### 7. IGRF pole
Pull IGRF-14 coefficients, compute magnetic coordinates properly. Pole drifts significantly; current hardcoded value will be wrong by 2030.

### 8. Module split
If the file grows beyond ~1000 lines, split into ES modules:
```
index.html
src/
  projection.js    # d3 setup, magnetic lat math
  data.js          # NOAA fetch + parse
  render/
    bands.js       # threshold zone rings
    ovation.js     # canvas aurora overlay
    cities.js      # city markers + labels
    pole.js        # magnetic pole marker
  ui/
    live-strip.js  # top bar with Kp + forecast
    sidebar.js     # solar wind, legend, city table
    toggles.js     # layer visibility buttons
```

Keep vanilla unless a real need drives a framework. No bundler yet — plain `<script type="module">` works fine.

## Design conventions

- Dark theme only. No light mode toggle.
- Tabular numerals everywhere numbers appear: `font-variant-numeric: tabular-nums`
- Borders are 0.5px. No thick lines, no drop shadows, no gradients.
- Muted palette: `--bg: #0A0E27`, `--ink: #E8EBF5`, accent teal `--accent: #7FE7C2`
- Sentence case for all text. No title case, no all caps except legend ticks.
- Numbers rounded for display — never leak float artifacts.
- Graceful degradation over fancy error states: if something fails, show the rest and keep going.

## Getting started

```bash
cd /Users/michaelpieniazek/Repos/solar
python3 -m http.server 8000
# then open http://localhost:8000
```

Or VS Code Live Server extension — right-click `index.html`, "Open with Live Server".

Chrome DevTools → Network → filter "services.swpc" to watch the NOAA fetches.

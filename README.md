# solar

Interactive aurora visibility globe. Live NOAA space weather data overlaid on a polar-projected Earth.

Single-page vanilla HTML/CSS/JS + D3. No build step. Drop it on any static host or serve locally.

## Run locally

```bash
python3 -m http.server 8000
open http://localhost:8000
```

Or use the VS Code Live Server extension. Direct `file://` opening will fail CORS on the NOAA fetches in most browsers — the page will detect this and show a hint.

## What it shows

- **Current Kp** — latest 3-hour observed geomagnetic index
- **72-hour Kp forecast** — NOAA SWPC predicted values
- **OVATION aurora nowcast** — real-time auroral oval as a probability grid, inferred from DSCOVR satellite particle measurements at L1
- **Solar wind** — speed, Bz, density from DSCOVR
- **Threshold zones** — static visibility bands (where aurora typically reaches at each Kp level), shown underneath the live overlay for reference
- **Cities** — 14 reference locations with their magnetic latitude and minimum Kp threshold

## Data sources

All open, no API key, CORS-enabled:

- NOAA SWPC: https://services.swpc.noaa.gov/
- Country topology: world-atlas via jsdelivr
- Magnetic pole position: approximate IGRF-2025

## Files

```
solar/
├── index.html    # the app (single file)
├── README.md     # this
└── HANDOFF.md    # architecture notes + extension ideas for a coding agent
```

## License

TBD.

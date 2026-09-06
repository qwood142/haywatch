# HayWatch 🌾

**Should I cut hay today - and will it dry before it rains?**

HayWatch is a hay dry-down forecast. Pick your crop and what you're making (small squares, round
bales, large squares, or wrapped baleage), and it scores the next 14 days by the quality of the drying
window that follows a cut - so you can pick a safe stretch to cut → ted → rake → bale.

It's a 100% client-side web app (installable PWA). No backend, no accounts, no API keys, no build step.
Everything runs in your browser off free public weather data.

## What it does

- **Cut score per day (0-100)** with a plain-language call: **Prime cut / Good / Marginal / Don't cut**,
  plus the single best day to cut in the next 14.
- **Multiple forecasts, cross-checked.** Rain risk is compared across several global weather models
  (ECMWF, GFS, ICON, GEM, Météo-France). When the models agree, confidence is High; when they split,
  it tells you to wait. That agreement is shown right on the forecast.
- **Drying power (reference ET₀)** - one number folding sun, heat, wind and dryness - drives the
  estimated dry-down days, adjusted for overnight dew and your cutting equipment.
- **Rain arrival timing** - not just "rain Wednesday" but *"heavy rain Wed from ~12 PM - you'd be
  baling into it,"* so you know your deadline to get off the field.
- **Cutting equipment** - mower-conditioner (crimped) vs plain cut. Crimping saves ~a day on
  thick-stemmed legumes (alfalfa, clover) and a little on grass, and the estimate reflects it.
- **Dew burn-off timing** - each day's morning dew-off and evening dew-set times: your workable window
  for cutting, raking and baling. Flags days when dew lingers.
- **Your plan for this cut** - a day-by-day cut → cure → bale timeline for the day you're eyeing.
- **Crop & making presets** - grass / mixed / alfalfa / clover / orchard-timothy × small squares /
  round bales / large squares / baleage. Wetter, wrapped baleage is far more rain-tolerant than dry hay;
  the model knows the difference.
- **Cutting log** - one tap to log a cut; set the outcome once baled (baled dry, rained on, made
  baleage) to track how the forecast actually held up on *your* fields.
- **Fields, location search, US/metric units, light/dark, offline, share card.**

## ⚠ Safety & disclaimer

Never bale or stack hay above safe moisture. Wet hay heats in storage and can spoil, mold, or
spontaneously combust - a real barn-fire risk. **Always confirm bale moisture with a probe before
storing.** HayWatch reads the weather forecast; it does not measure your hay.

HayWatch is a **planning aid, not a guarantee.** Forecasts are wrong sometimes, and only you can judge
your fields, equipment, and hay. **HayWatch is not responsible for any crop loss, spoilage, equipment,
property, or other damages arising from decisions made with this app.** Use your
own judgment - the weather gets the final say.

## Run locally

No build step. Serve the folder (the service worker and geolocation need HTTPS or localhost):

```bash
npx serve .
```

Then open the localhost URL. Opening `index.html` directly works for a quick look but disables the
service worker and geolocation.

## Deploy

Git-connected **Cloudflare Pages** project `haywatch` (repo `qwood142/haywatch`). Pages settings:
framework preset **None**, build command **empty**, build output directory **`/`** (root). Push to the
production branch and it auto-deploys.

## Data sources

- [Open-Meteo](https://open-meteo.com/) - forecast (incl. reference ET₀), multi-model precipitation, and
  place-name geocoding. Free, keyless.
- [FCC Area API](https://geo.fcc.gov/api/census/) - county/state for the location label.

## Tuning

Scoring weights (`CROPS`, `MAKING`, the rain/dew penalties) are informed guesses, not validated science.
The score is weather-driven and is not auto-adjusted by the log; the cutting log records how the
forecast actually panned out on your fields. See `CLAUDE.md` for the model details, schema, and roadmap.

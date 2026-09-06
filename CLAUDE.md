# CLAUDE.md — HayWatch

Project instructions for Claude Code. Read this fully before touching anything. HayWatch is a fork of
BiteWatch (fishing bite-time forecast) and shares its architecture and hard constraints.

## What this is

A hay dry-down forecast PWA. It answers **"when should I cut, and will it dry before it rains."** It
scores each upcoming candidate cut day by the quality of the drying window that follows, so a farmer
can pick a safe stretch to cut → ted → rake → bale. 100% client-side, no backend, no API keys, no
build step, no dependencies, no framework. The entire app is one file: `index.html` (HTML + CSS +
vanilla JS in a single `<script>`). This is intentional — see "Rules".

Home region: Potsdam / St. Lawrence County, NY (default coords 44.67, -75.00).

## Entity / naming

Legal entity: **Woods Market LLC**. HayWatch is a d/b/a (Certificate of Assumed Name) at the NY
Department of State — sibling to the BiteWatch d/b/a. Public face reads "HayWatch"; rolls up to the one
LLC. Git repo: `qwood142/haywatch` (created). Cloudflare Pages project: `haywatch` (created,
git-connected). Push to the production branch → auto-deploy.

## File map

```
index.html    the whole app (day aggregates, dry-down scoring, multi-model agreement, dew timing,
              cutting log, render, actions — all here)
manifest.json PWA manifest
sw.js         service worker (offline shell); bump CACHE const to force clients to refresh
README.md     user-facing deploy + tuning notes
CLAUDE.md     this file  ← source of truth for decisions/schema/tuning
icon-192.png / icon-512.png / icon-maskable-512.png   real HayWatch art (sun + round hay bale + grass
              tufts on the dark-green gradient). Source: haywatch_icon.svg.
haywatch_icon.svg     the icon source (512 SVG). Re-render the PNGs from it: open in the browser, draw to
              a canvas, toDataURL('image/png'); the maskable = art at 80% on a full-bleed #12271A bg.
              (No Node/Python on the owner's box; PNGs were rasterized via browser canvas + PowerShell
              System.Drawing. See session history if regenerating.)
.claude/launch.json   local dev-server config (npx serve on :3210) for the preview pane
```

There is no data file to load (unlike BiteWatch's `regs.json`) — everything comes from live APIs.

## Run locally

No build. Serve it (service worker + geolocation need HTTPS or localhost):

```
npx serve .
```

Or use the preview pane via `.claude/launch.json` (name `haywatch`). Opening `index.html` directly
(file://) works for a quick look but disables the service worker and geolocation.

## Deploy

Git-connected Cloudflare **Pages** project `haywatch`. Framework preset None, build command empty,
build output directory `/` (root). Push to the production branch → auto-deploy. Do NOT use the Workers
`wrangler deploy` create-flow for a static site — it deploys nothing without a config.

## Rules (do not break without an explicit decision from the owner)

1. **No backend, no server, no API keys, no build step, no npm dependencies.** Everything runs in the
   browser off free keyless APIs (Open-Meteo). Anything that breaks this is a deliberate v2 call.
2. **Single-file app.** Keep it in `index.html` unless the owner asks to split it.
3. **Logging stays one tap.** `Log a cut now` creates a complete record immediately; every extra field
   (outcome, actual dry days, moisture, note) is optional and defaulted. Never gate logging behind a form.
4. **Scoring weights are guesses, not truth.** Don't present them as validated. The cutting log is the
   calibration path. Preserve the "planning aid, not a guarantee" framing in the UI and docs.
5. **Safety + liability framing stays.** Hay baled wet is a real barn-fire / spoilage hazard. Keep the
   safety note (probe your bales) and the LLC liability disclaimer prominent. Privacy: LLC is home-address
   tied; no app stores. If catch/cut sharing lands (v2), the shared record fuzzes coords to field/area.

## Data (Open-Meteo, free, no key)

Two fetches per location, both keyless GETs:

- **Primary** (`loadWeather`): `forecast_days=14`, hourly `temperature_2m, relative_humidity_2m,
  dew_point_2m, precipitation, precipitation_probability, cloud_cover, wind_speed_10m,
  wind_direction_10m, vapour_pressure_deficit`; daily `precipitation_sum, precipitation_probability_max,
  et0_fao_evapotranspiration, temperature_2m_max/min, sunrise, sunset`. Requested in
  `temperature_unit=fahrenheit&wind_speed_unit=mph`; **precip and ET₀ stay in mm internally** and are
  converted for display.
- **Multi-model** (`loadModels`): hourly `precipitation` with `models=ecmwf_ifs04,gfs_seamless,
  icon_seamless,gem_seamless,meteofrance_seamless`. Per calendar day, counts how many models see
  meaningful rain (≥2 mm) → `rainFrac` / `rainCount`. This is the **forecast agreement** signal.
  ECMWF is short-range (~7 d), so far-out days naturally have fewer models = lower confidence.

`loadCounty` (FCC Area API) reverse-geocodes county/state for the location line. Open-Meteo geocoding
powers the place search.

## Scoring model (`scoreCut(D)` — the core new work vs BiteWatch)

Key driver: **`et0_fao_evapotranspiration`** (reference ET) is a single composite "drying power" number
folding in radiation, temperature, humidity and wind. Rain (amount × probability × model agreement) is
the dominant penalty.

For each candidate cut day D:
1. **`predictDryDays(D)`** — accumulate daytime effective ET₀ from D until it reaches the product's
   `needMm` (crop `dryFactor` × making `needMm`). Cut day counts as a half drying day. `effEt0` discounts
   ET₀ for genuinely saturated overnights via `dewMult`. Returns predicted dry-down days (or Infinity =
   not enough drying power in the horizon).
2. **Base** from drying-power surplus/deficit over the ideal window (`60 + 42·surplus`).
3. **Rain penalty (dominant):** over the curing window, `((amt·2.4)+(prob·6))·conf·qw / rainTol`, where
   `qw` rises across the window (rain on drier hay leaches quality worse) and `conf` discounts penalty
   when models disagree. A hard-rain day inside the window (≥7 mm with ≥40% model agreement, or ≥12 mm)
   forces the score to "Don't cut".
4. **Dew penalty:** small, only for RH ≥97% overnights (dew that lingers, not routine nightly dew).
5. **Short-window / not-enough-drying-power:** flagged and penalized; Infinity dry-down caps the score.
6. **Confidence** from model unanimity across the window × horizon decay → High / Medium / Low.

Tiers: **Prime cut ≥72 / Good ≥52 / Marginal ≥34 / Don't cut** (green/gold/orange/red). `bestCutDay()`
scans all 14 days for the top score. `riskLine(r)` emits the plain-language callout (e.g. "Heavy rain
Wed (4 of 4 models) — you'd be baling into it. Wait.").

### Crop / making / equipment presets (the "species" analog — calibrated guesses, log refines)

`dryNeed()` = `crop.dryFactor × making.needMm × (conditioned ? 1−crop.condBenefit : 1)`.

- **`CROPS.dryFactor`** (thick/waxy stems dry slower): grass 1.00, orchard/timothy 1.05, mixed 1.12,
  alfalfa 1.22, clover 1.25.
- **`CROPS.condBenefit`** (drying knocked off when cut with a mower-conditioner/crimper — big for
  thick-stemmed legumes, small for grasses): grass 0.06, orchard 0.08, mixed 0.14, alfalfa/clover 0.24.
- **`MAKING`** sets `needMm`, `maxDays` (window before "slow cure"), `rainTol` (rain forgiveness),
  `dewSens`, `wet`:

```
dry_square (small squares) needMm 8,  maxDays 4, rainTol 0.18   target 16–18%
dry_round  (round bales)   needMm 7,  maxDays 4, rainTol 0.32   target ~18%   (more rain-tolerant)
dry_large  (large squares) needMm 10, maxDays 5, rainTol 0.12   target ≤14%   (driest, most demanding)
baleage    (wrapped)       needMm 2,  maxDays 2, rainTol 0.70   target 45–55% (wilt ~a day; forgiving)
```

- **`CONDITIONED`** (persisted `hw_cond`, default true): mower-conditioner (crimped) vs plain cut. This
  is the "cutting equipment matters" dimension the owner asked for.

**Calibration (verified against extension sources + owner's lived NNY numbers):** conditioned grass
hay reaches baling moisture in ~2 days on a good stretch, ~3 mediocre, ~4 with heavy dew; conditioning
saves ~1 day on legumes; grass benefits little (thin stems, waxy cuticle). Baseline field-dry to hay
is 2–3 days (2–4 in tough weather). Sources: Purdue Pest&Crop (mower-conditioner adjustments), UGA
Forage ("Importance of Hay Conditioning"), UW-Madison Extension (field-drying forage — wide swath >
conditioning for fast dry-down; ≥70% cut width), Nebraska CropWatch / UMass (2–4 day alfalfa curing).
Still informed guesses — the cutting log's logged outcomes are the real calibration path.

**Predicted dry-down days** are shown as whole **calendar** days (`Math.ceil` of the ET₀ accumulator),
because that's how a farmer counts (cut Mon, bale Tue = "2 days").

### Rain-arrival timing (`rainArrival`, surfaced in the risk line + plan)

Per day, the first hour precip ≥0.4 mm or ≥55% chance = when rain arrives. The risk line uses it
("Heavy rain Wed **from ~12:00 PM** (4 of 4 models) — you'd be baling into it"), and the plan shows
"rain from ~3 PM" on wet days. This is the "will the front clip me, and by when must I bale" question.
Note: the global models are ~10–25 km grid, so exact-pin precision mostly matters via the near-term
high-res model already blended into `gfs_seamless` (HRRR, first ~18–48 h). A future win is adding HRRR
explicitly and weighting it for the 0–2 day window; wide-swath is another un-modeled fast-dry lever.

## Dew burn-off / set-in (`dewTimes`, surfaced in the plan + conditions)

Per day: morning **dew-off** = first hour after sunrise RH drops to ≤72% (swath workable); evening
**dew-set** = when RH climbs back ≥88% (stop baling). If RH never drops to 72%, dew **lingers all day**
(a poor drying day). Shown per day in "Your plan for this cut" and for today in Conditions. Currently
informational; wiring it into `effEt0` (shorten the effective drying day by the dew-off→set window) is a
reasonable future refinement.

## Cutting-log schema (v1) — sync-ready, same discipline as BiteWatch

```jsonc
{
  "id": "h...", "v": 1, "ts": 0,
  "lat": 0, "lon": 0, "field": null,        // nearest saved Field = per-field anchor (local only, fuzz before sharing)
  "crop": "grass", "product": "dry_square", // matches the preset picked
  "cutDate": 0,                             // the selected cut day (may be a future day, not just now)
  "ctx": { "et0Window": 0, "rainWindow": 0, "rainProbMax": 0, "minOvernightRH": 0,
           "score": 0, "predictedDryDays": 0 },   // forecast snapshot at log time
  "outcome": null,                          // baled_dry | rained_on | baleage | regrowth | (null = open)
  "moistureAtBale": null, "actualDryDays": null, "note": null   // all optional
}
```

Log stays one tap; every field after the snapshot is optional. `normalizeLog()` upgrades old records on
load/import. **The score is NOT informed by the log** (deliberate divergence from BiteWatch — owner
call): hay dry-down is weather/physics driven, so the forecast is the honest signal and auto-biasing it
from a handful of logged cuts would add noise. The log is a record + `calibration()` (how your calls
panned out, bucketed by the score at cut time — the honest-limits readout) + the seed for a future
shared per-field dataset. There is intentionally no `computeLearn`/score-feedback path.

## Storage

`Store` shim → localStorage with in-memory fallback. Keys: `hw_log`, `hw_fields`, `hw_lastloc`,
`hw_theme`, `hw_units`, `hw_crop`, `hw_making`, `hw_cond`. JSON helpers `jget`/`jset`.

## Panel order

Location-first by owner request (per-field weather is the priority in this app): **Field / location →
What you're making (crop / making / equipment) → best-day bar → today's call → 14-day outlook → plan →
drying-power chart → forecast models → radar (near your field, lazy) → conditions → cutting log →
How it works (FAQ)**. The Fields panel also carries a lazy "Compare my fields" table (≥2 fields).

## Forecast models panel + FAQ

- **Forecast models panel** (`renderModels`): for the SELECTED cut window, shows each model's total rain
  (from `MODELDAY[date].per[id]`) with a dry/wet dot — "shows what they show" so the user can cross-check
  sources the way they would toggle between apps manually. Intro line states agreement; **confidence is
  throttled by model count** (`modelFactor = min(1, avgCount/3)`) so far-out days where only 1–2 models
  reach never read as "High". Same `modelFactor` now applies to the hero confidence in `scoreCut`.
- **How it works (FAQ)**: native `<details>`/`<summary>` accordion (no JS), `.faq` styles. Covers score,
  the weather models (incl. the note that consumer apps repackage the same government models), ET₀,
  dry-down/conditioning, dew+rain timing, baleage, "does the log change the score? no", privacy/offline.

## Footer / privacy

The public footer disclaimer says "HayWatch is not responsible…" — the **LLC name is intentionally kept
out of public-facing text** (home-address-tied). NOTE: this repo deploys as static files, so dev docs
here (incl. the Woods Market LLC entity line above) are served publicly too, e.g.
`haywatch.pages.dev/CLAUDE.md`. If that matters, add an `.assetsignore` (`*.md`) — but Pages serves .md
by default and the owner accepts this on BiteWatch, so it's left as-is unless the owner says otherwise.

## Roadmap

1. ✅ v1: multi-model dry-down engine, crop/making/equipment presets (wet/dry, square/round,
   crimped/plain), region-calibrated dry-down (2/3/4 days), rain-arrival timing, 14-day outlook,
   drying-power + rain chart, cut→cure→bale plan, dew burn-off timing, cutting log with outcomes,
   location-first layout, Fields, PWA shell, share card, safety + liability framing.
2. Tip button URL (`TIP_URL` const at top of `<script>`, empty = hidden) — platform owner's choice.
3. ✅ Real HayWatch icon art (sun + round hay bale + grass on the dark-green gradient); source in
   `haywatch_icon.svg`.
4. ✅ **Scattered vs widespread rain classification.** `rainKind(d)` → `dry | scattered | widespread`
   from model spread (`rainFrac`), amount vs PoP, and convective-vs-stratiform WMO `weather_code`
   (`convSignals`: showers 80–82/95–99 = scattered; drizzle/steady 51–67/71–77 = widespread). A "40%"
   is coverage × confidence, so scattered = the "rains in Gouverneur, misses DeKalb" case. Surfaced in
   the risk line (`kindTail`: "widespread, don't count on dodging it" vs "scattered, your field may dodge
   it, but it's a gamble") and as a tag on the plan's wet-day rows. No new panel — folded into existing UI.
5. ✅ **Fields dashboard pieces (condensed).** (a) **Radar panel** — collapsible, lazy (loads only when
   opened); RainViewer (`api.rainviewer.com/public/weather-maps.json`, free/keyless/CORS-OK — tested).
   3×3 radar tiles at zoom 9 centered on the field, dark ground (precip only), SVG overlay with the field
   marker + 25/50 mi range rings, frame scrubber + play across ~2 h past (+ nowcast frames when present).
   `RADAR` state; `loadRadar`/`renderRadarFrame`; re-centers on field change if open. DISPLAY only — no
   pixel-sampling (CORS-dicey). Radar nowcast reaches ~0–60 min = same-day "keep-baling" mode, not
   planning. (b) **Compare my fields** — button in the Fields panel, shown only with ≥2 pinned fields;
   `compareFields()` lazily fetches each field (loadWeather+loadModels+buildDays), shows best-cut-day +
   next-rain(kind) per field in a compact table, then restores the current field's globals. This is where
   the DeKalb/Gouverneur split shows. **Deliberately CUT** the NWS 2.5 km / Open-Meteo `minutely_15`
   near-term layer as bloat — radar covers "is it coming" and multi-model covers planning. NWS
   (`api.weather.gov`, keyless, CORS-OK, ~2.5 km + alerts) remains a viable future add if wanted.
6. Other refinements: add HRRR (3 km) explicitly to the near-term rain agreement, weighted for the 0–2
   day window; add a wide-swath drying factor (extension: wide swath > conditioning for fast dry-down);
   wire dew-off→set window into `effEt0`; add `soil_moisture_0_to_7cm`; a cut/ted/rake/bale step tracker
   per active cut; push/rain-on-cut alerts.
7. v2 (deliberate, breaks no-server): shared per-field outcomes → a real dry-down dataset (same moat
   logic as BiteWatch). Cloudflare Worker + D1. Fuzz coords on the shared record.

## Known rough edges

- In a genuinely wet fortnight every cut window catches rain, so the whole 14-day outlook can read
  "Don't cut" with scores at 0. That's honest, not a bug — but the flatness hides relative ranking. If
  it bugs the owner, let the raw score go slightly negative internally so least-bad days still sort.
- Dry-down `needMm` and rain-penalty weights are unvalidated guesses; the log is the calibration path.
- The maskable icon has a faint seam where the 80% art tile meets the full-bleed background; invisible
  once a circular/rounded mask is applied. Fine for v1.

## Working style

Owner is direct and iterative: short prompts, prefers building over long explanation, auto-mode Claude
Code, reviews after; cross-checks data against local/first-party sources (defer to local data on
conflict). Keep this file current when decisions change or a session nears its limit.

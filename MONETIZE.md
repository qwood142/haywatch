# MONETIZE.md - HayWatch monetization game plan

Status: **planning only, nothing built.** Decide levers once there's real usage. This is the source of
truth for the monetization thinking; update it as decisions land.

## Why BiteWatch's model does NOT port

BiteWatch monetizes on **community / shared data** (the shared catch pool, per-water tuning) - a
consumer, social, network-effect play. HayWatch's audience is the opposite:

- **Fewer users**, utilitarian, B2B-ish (farmers, ranchers, custom operators, small livestock owners).
- **Each decision is worth real money** - a botched cutting is hundreds to thousands of dollars, plus a
  wet-bale barn-fire risk. High willingness-to-pay *per decision*.
- **Seasonal** - hay season is roughly May-Sept. Year-round subscriptions are a hard sell for a ~5-month
  tool.

So HayWatch should monetize on **decision value and timeliness**, not on a social data network. The
"gate community features behind a paywall" idea from BiteWatch is not applicable here.

## The ladder (matched to: solo dev, LLC, no backend today)

### Tier 1 - zero infrastructure, shippable now

- **Tip jar** - DONE in-app: expandable "Support HayWatch" popout, hidden until `TIP_URL` (top of the
  `<script>`) is set to a Ko-fi / Buy Me a Coffee / Stripe payment-link URL. Goodwill money; a farmer who
  dodges a rain-out will chip in. Owner just needs to pick a platform and paste the URL.
- **Affiliate links** - the app already tells people to probe bale moisture, ted the swath, watch for
  spoilage. Link a **moisture probe, hay tarps, tedder/rake parts** (Amazon Associates or direct
  merchant). Honest, contextual, passive. Best effort-to-revenue ratio right now. Natural spots: the
  safety note (moisture probe), the plan's "ted to speed drying" step, the FAQ.
- **One local sponsor slot** - a single tasteful "brought to you by [Farm Supply / Co-op]" line. A loyal
  niche utility is exactly what a regional ag retailer will pay a small monthly fee to sit in front of.
  Keep it to one, keep it clean (no ad networks - that would wreck the no-tracking privacy posture).

### Tier 2 - the real revenue engine (needs a light backend)

- **Paid alerts.** The feature farmers will actually pay for, and the natural premium tier:
  - "Rain headed for your field in ~2h - get off it" (rain-on-cut).
  - "A Prime cut window opens Thursday" (cut-window-opening).
  - Free app stays on-demand; paid unlocks proactive **Web Push** alerts (free to send, works from the
    installed PWA - no SMS cost). SMS/email optional later if wanted (SMS has per-message cost).
  - **Price seasonally** ($X for the hay season) - matches usage; avoids the year-round-subscription
    objection.
  - Infra (deliberately breaks the no-backend rule - a v2 call): **Cloudflare Worker + KV/D1** for
    subscriber state + a **cron** to evaluate each subscriber's fields and fire pushes; **Stripe** for
    the seasonal subscription/checkout. Store fields server-side (fuzz/limit coords per the privacy
    posture); VAPID keys for Web Push.
  - This is the one worth building a backend for - it's the line between "nice free tool" and "thing
    people pay for."

### Tier 3 - the data moat (later, B2B)

- Shared per-field cut **outcomes** (the cutting log is already schema-ready) -> a real regional
  dry-down dataset -> license / API to **co-ops, custom operators, extension offices, crop insurers**.
  Same "moat" logic as BiteWatch, but sold **B2B** instead of to consumers. Only matters once there are
  users and accumulated outcomes. Fuzz coords on any shared record (home-address-tied LLC).

## One-time "Pro" unlock (considered, parked)

Farmers prefer one-time over subscriptions, so a lifetime "Pro" unlock (gate radar / multi-field compare
/ alerts behind a license code) is tempting. Problem: **code validation is weak without a backend** -
an offline signed key is easily shared/pirated. Fine as an honor-system nicety, but not a serious
revenue mechanism on its own. If a backend gets built for alerts anyway, real licensing becomes easy -
so fold this into Tier 2 rather than doing it standalone.

## Recommendation

- **Now:** Tier 1 (tip URL + affiliate links + maybe one sponsor). Basically free to stand up.
- **When there's traction:** build **alerts** (Tier 2) - the real engine.
- **Later:** Tier 3 data/B2B once outcomes accumulate.

## Honest expectations

Niche + seasonal = modest revenue, not a SaaS rocket. The realistic shape is: a free tool that earns
goodwill and reach, with affiliate/sponsor trickle now and a small but real seasonal-alerts subscription
later. Per-user value is high; total user count is the limiter. B2B (co-ops/custom operators) is where
the bigger checks are if it ever grows.

## Open decisions for the owner

- Tip platform (Ko-fi vs Buy Me a Coffee vs Stripe payment link)?
- Affiliate program (Amazon Associates vs direct merchant relationships)?
- Willing to run a backend for alerts (breaks no-server), and at what season price?
- Comfort with a single sponsor line, and who?

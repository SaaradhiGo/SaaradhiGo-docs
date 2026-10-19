# Maps and pricing architecture

Written for: the engineers who will wire up a Maps provider and the person deciding
what it is allowed to cost.

Traced against the running code on 2026-10-19: backend `259e8d8`, web `110cc77`,
both live on QA. Nothing below is inferred from documentation — every row names the
file it was read from.

---

## 1. What actually exists today

The short version: **more is implemented than is switched on.** Maps is unconfigured
on QA (`GOOGLE_MAPS_API_KEY` unset), and the platform degrades honestly rather than
inventing data. Dispatch does not use Maps at all, which is the single most important
cost fact on this page.

| Capability | Where it lives | Provider | Triggered by | Cached | Billable | On failure |
|---|---|---|---|---|---|---|
| Destination search | `servers/ride/maps_utils.py:214` `google_places_autocomplete`, proxied by `servers/ride/views.py:1401` `proxy_geocode` | Places API — **New** (`places.googleapis.com/v1/places:autocomplete`) with a legacy path behind `GOOGLE_PLACES_API_VERSION` | Rider types a destination and taps Search | Redis, key `maps:geocode:<text>:<loc>`, **30 days** | Yes | 502 `API_ERROR` with the reason; the web offers manual coordinates, clearly labelled |
| Place details | `maps_utils.py:329` `google_place_details` → `views.py:1490` | Places New (`/v1/{place}`) or legacy | Rider picks a suggestion | Redis, `maps:place_details:<id>` | Yes | 502 `API_ERROR` |
| Geocoding | `maps_utils.py:428` `google_geocode` | Geocoding API | Address → coordinates | via the geocode proxy | Yes | 502 |
| Reverse geocoding | `views.py:1444` `proxy_reverse_geocode` | Geocoding API | "Use my location" | Redis, 5dp key (~1.1 m) | Yes | 502 |
| Route geometry | `maps_utils.py:462` `google_directions` → `views.py:1538` | Directions API | Trip tracking | Redis, `maps:directions:<o>:<d>`, **300 s** | Yes | 502 |
| **Fare distance** | `servers/ride/utils.py` `validate_distance` → `get_google_maps_distance` | **Distance Matrix** | **Every fare quote** | **Redis, 300 s — added this run** | Yes | Falls back to haversine × a per-zone road factor |
| Driver discovery | `servers/redis_client.py` (`GEOSEARCH` per vehicle-type key) | **Redis GEO — not Google** | Every dispatch wave | n/a | **No** | Empty candidate set |
| Dispatch / matching | `servers/ride/dispatch.py` — 3 waves, 1500/3000/5000 m, `DISPATCH_WAVE_SECONDS=20` | Redis GEO | Ride requested | n/a | **No** | `no_driver_accepted` |
| Live tracking / ETA | `DriverLocationConsumer`, Redis streams | Own GPS pipeline | Driver movement | n/a | **No** | Socket reconnect with backoff |
| Map rendering | `apps/web/src/components/MapSurface.tsx` | None configured | Page load | n/a | Would be | "Live map unavailable", booking unaffected |
| Navigation | Hand-off to the device's map app | n/a | Driver taps navigate | n/a | No | n/a |

**Tests already covering this:** `tests/test_maps_multi_country.py` (25),
`tests/test_distance_matrix_cache.py` (6, new this run).

### What is genuinely missing

- **No map tiles.** No basemap provider is wired anywhere (CARTO was evaluated, never configured).
- **No Routes API (New).** Directions and Distance Matrix are the legacy SKUs.
- **No waiting charges and no tolls** — no field, no code. `Trip.cancellation_fee` exists
  but `admin_dashboard/views.py:5268` records the deliberate decision: *"There is no
  cancellation charging in this platform."*
- **No autocomplete session tokens plumbed through**, so Places is billed per request
  rather than per session. At India rates per-request is the cheaper SKU, so this is
  currently an advantage, not a defect — revisit only if moving to Place Details Pro.

---

## 2. Cost model

Published prices below are Google's official list, retrieved **2026-10-19**, in **USD
per 1,000 billable events**. Everything in *italics* is a modelled assumption, not a
published figure.

### Published prices (India-specific SKUs, effective 2024-08-01)

India pricing applies to billing addresses in India with most usage in India, and it
is dramatically cheaper than global: **70,000 free events per Essentials SKU per
month** against 10,000 globally.

| SKU (Essentials) | Free / month | 70k–5M | 5M–10M |
|---|---|---|---|
| Geocoding | 70,000 | $1.50 | $0.38 |
| Autocomplete Requests | 70,000 | $0.85 | $0.21 |
| Place Details Essentials | 70,000 | $1.50 | $0.38 |
| Distance Matrix (legacy) | 70,000 | $1.50 | $0.38 |
| Directions (legacy) | 70,000 | $1.50 | $0.38 |
| Dynamic Maps | 70,000 | $2.10 | $0.53 |
| Static Maps | 70,000 | $0.60 | $0.15 |
| Maps SDK (mobile) | Unlimited | — | — |

Global list prices, for comparison: Geocoding/Distance Matrix/Directions $5.00,
Autocomplete $2.83, Dynamic Maps $7.00, all with only 10,000 free.

### Modelled request volumes per completed ride

*Assumptions.* `A` = booking attempts per completed ride (abandonment, no-driver,
cancellation). Expected **1.4**, conservative 1.2, high 1.8.

| Event | Per attempt | Basis |
|---|---|---|
| Autocomplete requests | *2.0* | one per Search tap; the web does not fire per keystroke |
| Place Details | *1.0* | one suggestion chosen |
| Reverse geocode | *0.2* | "Use my location", 30-day cache ⇒ high hit rate |
| **Distance Matrix** | *1.0* | **after this run's fix**; was *4.0* — one per vehicle category |
| Directions | *1.0* | route display, 300 s cache |
| Dynamic Maps load | *0.4* | web share only; mobile uses the free Maps SDK |

**Marginal cost per completed ride (expected, India rates, above all free tiers):**

```
autocomplete  2.0×1.4 = 2.8 × $0.85/1000 = $0.00238
place details 1.0×1.4 = 1.4 × $1.50/1000 = $0.00210
reverse geo   0.2×1.4 = 0.28× $1.50/1000 = $0.00042
distance mtx  1.0×1.4 = 1.4 × $1.50/1000 = $0.00210
directions    1.0×1.4 = 1.4 × $1.50/1000 = $0.00210
dynamic maps  0.4×1.4 = 0.56× $2.10/1000 = $0.00118
                                   TOTAL = $0.01028  ≈ ₹0.90
```

Before the Distance Matrix cache it was **$0.0166 (≈₹1.46)**. The fix removes ~38% of
marginal Maps spend.

### Monthly cost by volume (expected case, India rates, free tiers applied)

| Completed rides/day | /month | Monthly Maps cost | Per completed ride | Per booking attempt |
|---|---|---|---|---|
| 100 | 3,000 | **$0** (all SKUs inside free tier) | $0 | $0 |
| 1,000 | 30,000 | **≈ $12** | $0.0004 | $0.0003 |
| 10,000 | 300,000 | **≈ $2,435** | $0.0081 (₹0.71) | $0.0058 |
| 100,000 | 3,000,000 | **≈ $27,666** | $0.0092 (₹0.81) | $0.0066 |

Scenario spread at 10,000 rides/day: conservative ≈ $1,900/mo, expected ≈ $2,435/mo,
high (A=1.8, no Distance Matrix cache, half the cache hit rate) ≈ **$6,100/mo**.

At global rather than India pricing the 10,000/day figure is roughly **$9,700/mo** —
4× worse. Securing India pricing is worth more than every caching change combined.

**Biggest contributors at scale** (300k rides/mo): Autocomplete $655, Place Details
$525, Distance Matrix $525, Directions $525, Dynamic Maps $206. Autocomplete leads
because it fires *per attempt* and is the only one with a multiplier above 1.

### Savings already banked and still available

| Change | Status | Saving at 10k rides/day |
|---|---|---|
| Distance Matrix cached on the coordinate pair | **done** (`259e8d8`+) | ~$1,575/mo |
| Serve Directions from the existing 300 s cache more aggressively (30 min) | proposed | ~$400/mo |
| Static Maps instead of Dynamic for the non-interactive panel | proposed | ~$150/mo |
| Debounce + session tokens if Places moves to Pro | proposed | varies |

### Alternatives considered

- **MapLibre + self-hosted tiles** for rendering. Removes Dynamic Maps entirely
  (~$206/mo at 300k rides) for roughly $50–150/mo of tile-server infrastructure.
  Worth doing for the basemap; **not** for geocoding.
- **OSRM / Valhalla / GraphHopper** self-hosted for routing. Replaces Distance Matrix
  and Directions (~$1,050/mo at 300k rides) for ~$150–300/mo of compute, but owns map
  data updates and has no live traffic. Live traffic is what makes the *duration* half
  of the quote credible, so this is a hybrid candidate, not a wholesale replacement.
- **Nominatim / public OSM endpoints.** Rejected: the public services forbid this kind
  of commercial load, and the usage policy is not a rate limit we can buy our way out of.
- **Recommended hybrid:** Google for autocomplete, place details and traffic-aware
  distance (the things riders judge us on); MapLibre + self-hosted or commercial tiles
  for rendering; keep Redis GEO for dispatch, which already costs nothing.

---

## 3. Region-aware pricing — it already exists

`servers/pricing/models.py` already implements the full hierarchy the brief asks for.
**No second pricing engine is needed, and none should be built.**

`ServiceZone` carries: `country` (ISO-2), `state_code`, `city`, `zone_type` with a
self-referential `parent`, `polygon_geojson`, `priority`, **`currency`**,
**`timezone_name`**, **`distance_unit`**, **`tax_mode`**, `per_trip_fee`, `is_active`.

`RateCard` carries: `zone` × `vehicle_type`, `base_fare`, `per_km_fare`,
`per_min_fare`, `min_fare`, `night_surge_multiplier` with start/end hours,
`surge_cap_multiplier`, **`commission_percent`**, **`gst_percent`**, and
**`effective_from` / `effective_to` / `version`** for dated, audited changes.

Resolution is `find_zone_for_point()` by polygon and `priority`, so a city zone can
override a state zone without duplicating rates.

**Gaps against the target design, in priority order:**

1. **Commission lives only on the rate card.** A subscription model has nowhere to live.
   Add a `ZonePricingPolicy` (zone + effective dates + `model: commission|subscription|hybrid`
   + rate), read by `commission_percent_for_trip`. One engine, one new table.
2. **No waiting charges, no tolls.** Both are rate-card columns plus a term in
   `estimate_amount`.
3. **Regional payment providers** are not zone-driven; Cashfree is wired globally.
4. **No effective-dated audit surface in Admin** — the columns exist, the screen does not.

---

## 4. Backlog

**P0 — correctness, shipped this run**
- `259e8d8` trips record their vehicle; the rider's chosen category is exposed.
- Driver counter drift: full `driver.save()` from a connect-time snapshot (below, §5).
- Distance Matrix fan-out: 4 billable calls per quote collapsed to 1.

**P1**
- `ZonePricingPolicy` for commission vs subscription.
- MapLibre basemap so the map stops being a blank panel.
- Lengthen the Directions cache; Static Maps for the summary panel.
- Derive `total_trips` at read time, or reconcile it nightly (see §5).

**P2**
- Waiting charges and tolls.
- Routes API (New) migration off the legacy Directions/Distance Matrix SKUs.
- Places session tokens, if and only if Place Details moves to Pro.

---

## 5. The counter drift, for the record

`Driver.total_trips` read 3 against 4 authoritative completed trips. The increment was
never the problem: it is a database-side `F('total_trips') + 1` inside the transaction
already holding the trip row, and the transition guard means a replayed completion
returns `already_in_target_state` without reaching it.

`DriverLocationConsumer.connect()` loads `self.driver` once and holds it for the life of
the socket. Going online or offline then called a bare `self.driver.save()` — every
column, from a snapshot that could be hours old. A trip completing mid-session moved the
database row; the next toggle wrote the stale row back over it.

`total_trips` was the visible casualty. The same write reverted anything else changed
during the session — `ratings`, `approved`, `qa_simulated_eligibility`, `active_vehicle`.
**An operator approving a driver's KYC while that driver was online would have had the
approval silently undone the moment the driver went offline.**

Fixed by scoping both saves to `update_fields=['status']`. Reproduced first, then fixed:
`tests/test_driver_counter_drift.py` fails with `assert 0 == 4` against the old code.

**Recommendation:** keep `total_trips` as a stored counter — it is read on hot paths and
deriving it would add a `COUNT(*)` to every profile load — but treat it as a cache, not a
ledger. Money must never be derived from it (it is not today: earnings come from the trip
and payment tables, which is why they stayed correct throughout). Pair it with the
existing `recompute_driver_trip_counts` run nightly as reconciliation.

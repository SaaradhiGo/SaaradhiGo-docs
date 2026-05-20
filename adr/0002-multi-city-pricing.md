# ADR-0002: Multi-city pricing with versioned, effective-dated rate cards

* Status: Accepted
* Date: 2026-05-20
* Deciders: Architect, Founder
* Supersedes: none
* Superseded by: none

## Context

Phase-0 ships with a single-city, hand-edited fare table
(`servers/ride/models.py:VehicleFarePricing`) and a hardcoded
Hyderabad polygon in `base/service_area.py`. This was the fastest
path to launch, but it has three structural problems we will hit on
the way to Phase-1:

1. **Service area is code.** Adding Bangalore today requires editing
   a Python module, running migrations, and redeploying. Ops cannot
   onboard a new city without engineering.
2. **Pricing is not versioned.** Editing a `VehicleFarePricing` row
   takes effect immediately, with no rollback path and no audit
   trail of "what was the fare schedule when this trip ran?". This
   becomes a real problem the first time a regulator or a rider
   disputes a fare months after the trip.
3. **Pricing is global per vehicle type, not per zone.** Bangalore
   and Hyderabad will have different base fares (different fuel,
   different competitive landscape, different MV Department caps).
   The current schema cannot represent that.

We also explicitly do **not** want an ML pricing algorithm for
Phase-1. India's Motor Vehicles Aggregator Guidelines 2020 cap
dynamic surge at 1.5x normal fare and require an explainable fare
formula -- a black-box ML price you cannot justify to a transport
commissioner is a non-starter. ML enters in Phase-3 (Q1 2027) after
≥6 months of trip data, and even then as an input *to* the rule
engine, not a replacement.

## Decision

Introduce a new Django app `servers.pricing` with two tables:

### `ServiceZone`
Polygon-on-the-globe where SaaradhiGo operates. Hierarchical
(country → state → city → sub-zone) so an airport sub-zone can
inherit fare rules from its parent city. Polygon stored as GeoJSON
in a JSONField -- **no PostGIS dependency** for Phase-1 -- with
point-in-polygon evaluated in-process via Shapely. Polygons are
cached in module memory and invalidated by a `post_save` signal so
ops edits take effect immediately. Higher-priority zones win when
multiple zones contain the same point.

### `RateCard`
Versioned, effective-dated fare schedule per
`(zone × vehicle_type)`:

* `base_fare`, `per_km_fare`, `per_min_fare`, `min_fare`
* `night_surge_multiplier` + `night_surge_start_hour` /
  `night_surge_end_hour` (configurable per zone -- night windows in
  Mumbai differ from Hyderabad)
* `surge_cap_multiplier` (MVA 2020 default 1.50; per-zone so other
  state caps can be honoured)
* `commission_percent` and `gst_percent` (informational; payments
  service is authoritative for the actual settlement)
* `effective_from` / `effective_to` + `version`

Lookup contract: at any timestamp T there is **at most one** active
card per `(zone, vehicle_type)`. Scheduling a future rate change is
a data operation -- insert a new card with a future `effective_from`
and the resolver picks it up automatically. Rollback is also a data
operation -- set `effective_to=now()` on the bad card; the resolver
falls back to the previous one.

If a sub-zone has no card for a vehicle type, resolution walks UP
the `parent` chain so the city card serves as the default.

### Pricing service

The single entry point for fare quotes is
`servers.pricing.services.quote_fare(distance_km, duration_min,
vehicle_type, pickup_lat, pickup_lon, rider_id)`. It composes:

1. `find_zone_for_point(lat, lon)` -- most specific active zone
2. `get_active_rate_card(zone, vehicle_type)` -- with parent
   fall-through
3. `compute_surge_multiplier(...)` -- the same rule-based engine
   from `servers/ride/utils.py`, extracted into the pricing service
   so every caller surfaces the same number
4. Night surge if the local hour is in the card's window
5. Surge cap enforcement (compound surge can never exceed
   `surge_cap_multiplier`)
6. Min-fare floor
7. A `{total_fare, breakdown..., zone_code, rate_card_version,
   source: 'db'|'default'}` dict so receipts/audits can prove which
   card priced which trip

### Backwards compatibility

* `base/service_area.py` becomes a thin shim that delegates to
  `find_zone_for_point` and falls back to the legacy in-memory
  Hyderabad polygon if no zones are configured (so a fresh dev DB
  still works).
* `servers/ride/utils.py:estimate_amount` becomes a thin wrapper
  around `quote_fare` returning the exact dict shape existing
  callers expect, plus two new keys (`zone_code`,
  `rate_card_version`).
* `VehicleFarePricing` is kept in place for one phase. The 0002
  data migration ports every existing row into a Hyderabad-scoped
  `RateCard`. A future ADR will retire `VehicleFarePricing` once we
  confirm no callers remain.

### Admin surface

Two admin-only `ModelViewSet`s under `/api/v1/pricing/admin/`:

* `admin/zones/` -- ServiceZone CRUD with GeoJSON polygon
  validation (closing-point check, lon/lat range)
* `admin/rate-cards/` -- RateCard CRUD, filterable by zone /
  vehicle_type / is_active, plus an `effective/` action that
  returns "which card is currently in force for (zone, vt)"

Public list of active zones (`/api/v1/pricing/zones/`) for the
rider/driver app's "we serve these areas" page. Public quote
endpoint (`/api/v1/pricing/quote/`) is identical to the existing
`/api/v1/ride/estimate-fare/` but reads coordinates only -- distance
is recomputed server-side.

## Consequences

### Positive

* **Adding a new city is data, not code.** Ops inserts a
  `ServiceZone` row + N `RateCard` rows in the admin UI. No deploy.
* **Pricing changes are auditable + reversible.** Every trip can be
  re-priced from its stored `(zone_code, rate_card_version)` even
  years later.
* **Surge cap is enforced centrally.** The pricing service refuses
  to return a multiplier > `surge_cap_multiplier`, so we cannot
  accidentally violate MVA 2020 from a runaway dynamic-surge rule.
* **Sub-zones unlock airport / event / station pricing** without
  duplicating the whole city schedule.
* **Receipts get richer.** `zone_code` and `rate_card_version` flow
  into FarePricing / Trip records so a dispute can be settled by
  pointing to the exact card.

### Negative / costs

* **One more app + ~700 lines of code** to maintain. Mitigated by
  the fact that it absorbs logic that already existed in two
  separate places (`base/service_area.py`, `ride/utils.py`) and
  centralises it.
* **Shapely in-process polygon checks** are O(zones) per lookup.
  For Phase-1 (<20 zones) this is single-digit microseconds; if we
  cross ~100 zones we move to PostGIS + a spatial index. Migration
  is straightforward because the polygon is already GeoJSON.
* **Operators must learn the rate-card UI.** Mitigated by Django
  admin + a one-page ops runbook (TBD).

### Neutral

* **No PostGIS dependency added.** We chose Shapely + JSONField
  deliberately to keep the Phase-1 ops footprint small. The model
  is forward-compatible with PostGIS -- when we need it, the column
  can be migrated to `PolygonField` with a single data migration.
* **No ML.** Surge stays rule-based. Re-evaluate in Phase-3 after
  ≥6 months of trip data, and only as a *signal* feeding the rule
  engine.

## Alternatives considered

1. **Just add a `zone` column to `VehicleFarePricing` and a
   separate `City` table.** Cheaper to implement but does not solve
   versioning, doesn't give us audit trails, and ties the
   service-area polygon to a per-row column instead of a first-class
   model. Punted.

2. **Use PostGIS from day one.** Cleaner long-term but adds an ops
   dependency (PostGIS-enabled Postgres image, GIS-aware backups,
   PostGIS-version compatibility on RDS). For ≤20 zones the
   in-process Shapely check is fast enough; revisit at Phase-3.

3. **Build an ML pricing engine now.** Rejected -- no historical
   data, no conversion baseline, and the MVA 2020 cap requires an
   explainable fare formula. Premature ML is a classic waste.

4. **Per-city Django setting.** A flat config dict per city is
   marginally simpler but loses versioning, audit, and any sub-zone
   concept. Same trap as the current `VehicleFarePricing` table,
   just spread across more cities.

## Implementation

* Backend: branch `feat/multi-city-pricing`, single PR against
  `dev`. Touches `servers/pricing/` (new), `base/service_area.py`
  (now a shim), `servers/ride/utils.py:estimate_amount` (now a
  wrapper), `base/settings.py` (INSTALLED_APPS), `servers/urls.py`
  (route inclusion). Migrations:
  `pricing/0001_initial.py`, `pricing/0002_seed_phase0_data.py`
  (ports `VehicleFarePricing` to `RateCard`, or seeds Phase-0
  defaults if the legacy table is empty).
* Tests: `tests/test_pricing_multi_city.py` -- 16 cases covering
  zone lookup, rate card resolution, parent fall-through, quote
  end-to-end, admin RBAC, polygon validation.
* Docs: this ADR, plus a cross-link from
  `business/pricing-phase-0.md` pointing to the new admin UI
  instead of the SQL seed snippet.

## Follow-ups (out of scope for this ADR)

* ADR-0003: deprecate + drop `VehicleFarePricing` once no callers
  remain.
* Runbook: "Onboard a new city" (insert zone -> insert rate cards
  -> smoke-test `/api/v1/pricing/quote/`).
* Phase-2: add a `SurgeRule` table so the tiered surge thresholds
  themselves are editable per zone.
* Phase-3: ML-based demand forecasting as a *signal* feeding the
  rule engine, never replacing it.

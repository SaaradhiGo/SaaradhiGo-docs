# ADR-0006: Phase-0 launch expansion to Vijayawada, Warangal, and Visakhapatnam

* Status: Accepted
* Date: 2026-05-22

## Context

[ADR-0002](0002-multi-city-pricing.md) introduced ServiceZone + RateCard so adding a city is a data operation. This ADR records the **first** use of that mechanism: launching three additional Phase-0 cities alongside Hyderabad.

The cities chosen are AP / Telangana metros within day-trip distance of the Hyderabad ops hub, sharing the same DPDP, RBI, MVA 2020, and GST regime, and avoiding the regulatory complications of crossing into a state where we don't already have a transport-department dialogue.

## Decision

Activate three new ServiceZones at launch:

| Code | City | State | Approx. centroid | Tier | Rationale |
|---|---|---|---|---|---|
| `IN-AP-VJA` | Vijayawada | Andhra Pradesh | 16.51 N, 80.63 E | 2 | Largest metro in AP after Vizag; commercial hub. |
| `IN-TG-WGL` | Warangal | Telangana | 17.99 N, 79.59 E | 2 | Strong intra-city auto demand; small enough to validate Tier-2 pricing. |
| `IN-AP-VTZ` | Visakhapatnam | Andhra Pradesh | 17.69 N, 83.21 E | 1 | Coastal Tier-1; longer airport corridor, port traffic. |

### Pricing variants

Phase-0 schedule mirrors Hyderabad's structure with regional adjustments:

| Vehicle | HYD base | VJA base | WGL base | VTZ base |
|---|---|---|---|---|
| auto | Rs 30 | Rs 25 | Rs 25 | Rs 30 |
| hatchback | Rs 45 | Rs 40 | Rs 40 | Rs 45 |
| sedan | Rs 60 | Rs 60 | Rs 55 | Rs 60 |
| suv | Rs 100 | Rs 100 | Rs 95 | Rs 105 |

Per-km, per-min, min-fare, surge cap (1.5x MVA 2020), night surge window (11pm-5am), commission (18% auto/hatchback, 20% sedan/suv), and GST (5%) are all set per zone via `RateCard`. Concretely codified in the data migration `servers/pricing/migrations/0004_seed_vja_wgl_vtz.py` and re-confirmable by reading the `pricing_ratecard` table after migration.

### Polygons

Each zone's GeoJSON polygon is a generous convex hull around the metro core + airport corridor. Coordinates are committed to the migration so a fresh DB reproduces them. Tightening or correcting a polygon is a Django-admin operation -- no code change required.

### Operations

Each new city requires the same Phase-0 founder actions as Hyderabad before public launch:

* State RTO aggregator licence dialogue (Telangana already covers HYD + WGL; AP RTO needed for VJA + VTZ).
* GST place-of-supply mapping (separate state codes for AP).
* TDS u/s 194O accumulates by PAN, not by city -- no city-level change.
* First 30-50 drivers per city sourced + onboarded per [legal/mva-2020-driver-verification-sop.md](../legal/mva-2020-driver-verification-sop.md).
* Local customer-support phone + WhatsApp staffed during operating hours.

## Consequences

### Positive
* The platform now operates in four cities, in two states. Existing pricing, surge cap, KYC, SOS, refund, receipts, wallet, notification, and admin surfaces all just work via zone lookup -- no code change.
* The activation set up the **template** for future cities: insert one ServiceZone + N RateCards.
* Tier-2 pricing variants give us actual Phase-0 data on whether the lower-base hypothesis is correct.

### Negative
* Larger ops surface: four cities of driver supply to source + manage, four cities of customer support to staff, two state RTOs to maintain a dialogue with.
* Inter-state GST place-of-supply rules need a CA review the first month so we don't mis-file.
* If demand is lopsided (e.g. all VTZ, no WGL), we'll need to flip `is_active=False` on the under-performing zones rather than thrash the team.

### Neutral
* Surge engine is per-zone already; no change.
* Withdrawal flow is identical (Cashfree Payouts for drivers anywhere in India).

## Alternatives considered

1. **Stay Hyderabad-only for 90 days.** Cleaner data, slower growth. Rejected because the multi-city pricing infrastructure exists and the marginal ops cost of two adjacent metros is small enough to justify.
2. **Add Bangalore + Chennai + Mumbai instead.** Larger metros, much larger ops + regulatory lift, different state RTOs we don't yet have relationships with. Punted to Phase-1.
3. **One city at a time with a 30-day soak.** Safer but slower. The Hyderabad pilot is the soak; these three are launching after we've already absorbed the Hyderabad operational lessons.

## Implementation

Single data migration `servers/pricing/migrations/0004_seed_vja_wgl_vtz.py` seeds the three ServiceZones + 12 RateCards (3 cities x 4 vehicle types). Tests in `tests/test_phase0_extras.py` assert zone lookup resolves coordinates inside each polygon and that quote_fare returns the city-specific rate.

## Follow-ups

* Per-city KPI breakouts in the ops dashboard (currently aggregated).
* Per-city promo-code campaigns (PromoCode.zone is already in place).
* Per-city support routing (assigned_to + region tag on SupportTicket).
* Phase-1 expansion to Bangalore + Chennai + Mumbai once the four-city operation is steady.

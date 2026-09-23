# ADR-0011 (DRAFT) — Observe-only trip actuals and fare shadow

- **Status:** DRAFT. Implemented as observe-only; no monetary behaviour changed.
- **Date:** 2026-09-23
- **Depends on:** [ADR-0010](0010-DRAFT-trip-gps-trail.md) (durable GPS trail)
- **Related:** [fare-finalisation-design](../runbooks/fare-finalisation-design.md)

## Context

The rider is quoted up front, `final_fare` is never written, and the platform had
no measurement of what a trip actually was: no distance, no real duration, no idea
whether metering would collect more or less. Every argument about fare
finalisation was therefore an argument about opinions.

The durable trail (ADR-0010) made measurement possible. This ADR records the three
decisions taken to turn it into evidence without touching anybody's money.

## Decision 1 — the passenger journey is `started_at → completed_at`, proven from OTP gating

Billable distance must cover the passenger's journey, not the driver's approach to
the pickup. The boundary was derived from the lifecycle rather than assumed:

* `in_progress` is the **only** transition gated on the rider's OTP
  (`_update_trip_status('in_progress', otp_input=...)` compares `trip.otp`), and
  the OTP is the rider's secret read aloud to the driver at pickup. A successful
  verification is therefore evidence the rider is physically present.
* `in_progress` sets `started_at`. `reached` sets `reached_at`, which is the driver
  *arriving* with the rider not yet aboard.
* The transition table allows `completed` only from `in_progress`.

So `accepted_at → started_at` is unbilled approach, and points outside the journey
window are excluded. A test loads 20 approach points and 4 journey points and
asserts only the journey is measured: if this were wrong, every rider would pay for
the driver's drive to them.

## Decision 2 — duration from timestamps, distance from GPS, coverage always reported

**Duration** comes from the lifecycle timestamps, not from summing GPS intervals.
They are authoritative server-side facts and are always present on a completed
trip. Summing intervals would under-report duration exactly when telemetry is
worst — the opposite of what a billing input should do.

**Distance** has no such authority available, so it is summed from the trail. Two
consequences are accepted rather than hidden:

* Every result carries `coverage_ratio`, `max_gap_seconds` and
  `rejected_segments`. A distance from 40% coverage is not the same fact as one
  from 95%, and the fare policy must be able to tell them apart.
* `distance_km = None` (unknown) is kept distinct from `0.00` (did not move).
  Collapsing them would let a future metered fare bill a real journey as nothing.

A segment implying more than 150 km/h is excluded as a GPS jump rather than summed;
one cold-start fix in the wrong cell would otherwise add tens of kilometres to a
1 km trip.

**No external maps API.** Fare completion must never depend on Google or Mapbox
availability, so the calculation is local, deterministic and replaceable by
swapping one function.

## Decision 3 — the shadow quotes both legs at one instant, and cannot change a price

`quote_fare` reads the live surge multiplier, the currently effective rate card and
the wall clock for night surcharge. Comparing a fresh actual-distance quote against
the historical `estimated_fare` would blend the distance difference together with
hours of pricing drift and produce an uninterpretable number.

So the shadow re-quotes the **estimate** alongside the **actual**, in the same
instant and the same pricing context, and stores the two differences separately:

```
shadow_actual - shadow_estimate   -> distance and duration alone
shadow_estimate - quoted_fare     -> the pricing context having moved
```

Both quotes pass `record_demand=False`. A shadow quote that registered demand
would raise the live surge multiplier for real riders — a shadow changing prices,
invisibly. This is asserted on every call in a test rather than trusted.

Every amount comes from the canonical `quote_fare`. A second fare formula for
shadow purposes would eventually measure a fare nobody charges.

## Consequences

* `Trip.actual_distance_km` and `actual_duration_min` are populated for completed
  trips, via a narrow `update_fields` that cannot touch `final_fare`. A test
  changes `final_fare` behind a stale instance and asserts it survives.
* `TripFareShadow` holds one row per trip. No payment, wallet, commission or
  settlement path reads it, and nothing in the fare path writes it.
* Trips that cannot be measured or shadowed are recorded **with the reason**, so
  any later "metering would collect X% more" has a visible denominator.
* Both are delivered out of band: actuals on a delayed Celery task after commit
  (the trail drains on a schedule, so computing at completion would under-read
  distance), the shadow on a bounded 15-minute sweep. Neither can slow or fail
  trip completion.
* The shadow is off by default (`FARE_SHADOW_ENABLED`), so it deploys before it
  observes.

## What this deliberately does not do

It does not write `final_fare`, add `fare_basis`, change what a rider is charged,
change commission, or change settlement. Those decisions — and the eight
commercial ones the shadow exists to inform — are listed in
[fare-finalisation-design](../runbooks/fare-finalisation-design.md).

## Status note

DRAFT until the shadow has run against real traffic and the fallback bands in the
fare-finalisation design are chosen from that data. The decisions above are not
expected to change; the thresholds derived from them are.

## Defects this work uncovered

Both were found by testing the **joins** over real infrastructure, not by testing
the rules. Neither was visible to any unit test.

**The trail discarded the end of every journey.** The writer resolved a ping's
trip from the driver's status *at drain time*. The drain is periodic, so by the
time it runs the pings from the end of a journey belong to a trip that has already
completed — and they were dropped as "no active trip". Every measured distance was
short by however far the driver travelled since the previous drain, and a lagging
or restarted worker would lose an entire trail, making the trip read as having no
telemetry at all. Fixed by matching each ping's `recorded_at` against each trip's
collection window, with a bounded grace period for trips that have just ended.
Two further properties came with the fix: a ping recorded before the driver was
assigned is no longer attributed to the trip they later accepted (it was storing
an idle driver's private movements), and a driver already on their next trip no
longer has the previous trail merged in.

**Surge demand silently counted zero.** `count_nearby_active_riders` passed
`Decimal` coordinates to redis-py, which rejects them. The exception was caught
and the function returned 0 — which is also a legitimate answer — so the only
evidence was an ERROR log line. It surfaced because the fare shadow calls the
canonical quote with coordinates read off a `Trip` row. `nearby_drivers`, which
the whole dispatch pipeline depends on, had the same latent defect and was safe
only because one call site remembered to wrap its arguments in `float()`. Fixed at
the Redis boundary.

The pattern is worth stating: the earlier wrong-Redis-database defect in this same
writer, and both of these, were each found by a test that used real Redis and real
PostgreSQL rather than mocks. Mocked tests prove the policy. Only the real thing
proves the plumbing.

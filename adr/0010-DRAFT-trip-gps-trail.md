# ADR-0010 (DRAFT): durable trip GPS trail

* Status: **DRAFT — substrate implemented, live integration deliberately deferred**
* Date: 2026-09-22
* Branch: `feat/trip-gps-trail` (unmerged)
* Relates to: [ADR-0008](0008-durable-celery-dispatch.md)

## Context

Driver location is Redis-only: a per-vehicle-type GEO index for matching, a
45-second heartbeat, and an ephemeral `driver_location_stream`. Nothing
survives, so a completed trip leaves **no route behind**. That single gap blocks:

* actual distance and duration, and therefore fare reconciliation;
* "the driver took a longer route" disputes;
* SOS and safety investigation, and any regulator or police request;
* per-trip insurance evidence;
* any future ETA or surge modelling, which needs history that is not being
  collected and cannot be backfilled.

## Decision

Add `TripLocationPoint` as the durable half. Redis stays the live answer to
"where is this driver now"; PostgreSQL becomes history.

**Write path uses the stream that already exists.** `update_driver_location`
already does `XADD driver_location_stream` on every ping. A batched Celery
consumer drains it, samples, and bulk-inserts. So the per-ping hot path — sized
for 1000 pings/second — takes **no additional database work at all**, and
persistence failure cannot affect live location.

```
driver app --ping--> Daphne --> Redis GEO               (live matching, unchanged)
                            \-> driver_location_stream  (already exists)
                                        |
                        Celery drain (batched) --sample--> TripLocationPoint
```

### Schema

`trip`, `driver`, `latitude`/`longitude` (Decimal 10,7 matching
`Trip.pickup_lat`), `recorded_at` (device clock), `received_at` (server clock),
`accuracy_m`, `speed_kmh`, `heading_deg`, `sequence`, `source`.

Two design points worth stating:

* **Both clocks are kept, because they disagree.** Phone clocks drift by
  minutes. `recorded_at` is untrusted and used for ordering; `received_at` is
  ours and is what anything financial or legal must reason about.
* **`driver` is denormalised from the trip on purpose.** Safety and insurance
  queries ask "where was driver X at time T" without knowing the trip.

Indexes: `(trip, recorded_at)` for route replay and distance,
`(driver, -recorded_at)` for SOS/insurance, `(received_at)` for retention
sweeps. A unique constraint on `(trip, recorded_at, sequence)` refuses an exact
re-send.

### Sampling — the load-bearing decision

Persisting every ping would be ~1000 inserts/second of largely redundant data;
a driver idling at a pickup emits the same coordinate repeatedly. Four filters,
cheapest first:

| Filter | Threshold | Purpose |
|---|---|---|
| Validity | parseable, in range, not `0,0` | `0,0` is the classic "no fix" sentinel |
| Accuracy | drop if over 50 m | a 500 m fix next to a 5 m fix makes derived distance *worse* |
| Time | at least 5 s since last kept | one point per 5 s reconstructs a city route |
| Distance | at least 25 m since last kept | this is what stops a stationary driver filling the table |

Endpoints are always kept — first point of a trip, and any explicitly final
point — because otherwise a short trip can reduce to one point and its distance
becomes unrecoverable. Hard cap of 5000 points per trip as a broken-client
guard (about 7x a 60-minute trip at 5 s).

Measured effect in tests: a stationary driver emitting 50 pings yields **1**
stored point; a moving driver yields all 10.

### Collection window

`trip_is_collecting()` reads the durable status and reuses
`DRIVER_ACTIVE_TRIP_STATUSES` — the *same* constant as the driver active-trip
invariant. Collection therefore starts when a driver is committed to the trip
(`accepted`) and stops the instant it becomes terminal, with **no separate flag
to drift**. A `requested` trip has no driver and so no track.

## Privacy and retention

Location history is sensitive data and is treated as such.

| Concern | Decision |
|---|---|
| Collection start | trip enters `accepted` |
| Collection stop | trip becomes `completed` or `cancelled`, enforced by reading PostgreSQL rather than a flag |
| Rider location | **never stored.** No rider FK and no rider coordinate exists on the model; a test asserts this |
| Logging | coordinates are never logged. `location_trail` logs counts and trip ids only; a test greps its own source to enforce that, and `PIIRedactionFilter` redacts lat/lng-shaped keys as a second line |
| Access control | not exposed on any rider or driver API in this change. A driver must not read another driver's track, and a rider must not read the driver's track beyond their own trip |
| Admin visibility | not registered in Django admin in this change — deliberate, so a browsable full-fleet location history does not appear by default |
| Retention | **recommendation, not implemented:** 90 days for ordinary trips; retain longer only for trips with an SOS event or an open dispute. Deletion is a data-policy decision, so no sweeper is shipped |
| DPDP | `/me/delete` anonymisation must be extended to cover this table before it carries real data |

## What was implemented, and what was not

**Implemented** (branch `feat/trip-gps-trail`, unmerged): the model, migration
`ride/0012_triplocationpoint`, the sampling/validation service, and 38 tests.
Full suite 385 passed on SQLite and on PostgreSQL 15.

**Deliberately NOT implemented: the stream-drain writer.** It touches the
location hot path, and getting sampling or batching wrong there creates write
amplification against the busiest code path in the system. The
expensive-to-change part — the schema — is landed and reviewable; the
integration is a small, separately reviewable step that deserves full attention
rather than the end of an overnight pass.

Remaining work, in order:

1. Celery task draining `driver_location_stream` in batches, resolving each
   driver's active trip from PostgreSQL, sampling, and `bulk_create`.
   Idempotency comes from the unique constraint plus stream ids.
2. `actual_distance_km` / `actual_duration_min` derivation from the trail —
   this is the input fare finalisation needs (see the fare report).
3. Extend `/me/delete` anonymisation to this table.
4. Retention sweeper, once the retention policy is agreed.
5. Authorisation tests for any read API, when one is added.

## PostGIS

Not required for MVP and not adopted. `Decimal` lat/long with a haversine
threshold is sufficient for sampling and for distance summation, and the table
is designed so a later `PointField` can be added alongside and backfilled
without changing the read patterns. Adopt PostGIS when a genuine spatial
*query* appears — "which trips passed through this polygon" — not for storage.

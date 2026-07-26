# ADR-0007: Pre-launch hardening — dispatch, settlement, and trip-transaction boundaries

* Status: Accepted
* Date: 2026-07-25
* Deciders: Architect, Founder
* Supersedes: none
* Superseded by: none

## Context

A pre-launch architecture review of all four repos found a set of defects
that share one root cause: **the trip lifecycle was written as one long
synchronous function**, and everything the platform needed to do — notify,
settle, invoice, dispatch, index — was bolted into it wherever it was
convenient rather than at a boundary chosen on purpose.

The specific findings, all confirmed in code on `dev`:

1. **Trip completion held a row lock across three external systems.**
   `_update_trip_status` opened `transaction.atomic()`, took
   `SELECT FOR UPDATE` on the `Trip` row, and inside that lock rendered a
   PDF receipt (reportlab), uploaded it to S3, sent it via SES, and created
   a Cashfree order. Trip-completion latency was therefore a function of
   SES availability. A rollback after the gateway call left an order at
   Cashfree with no local row.

2. **Any approved driver could subscribe to any unassigned trip** and
   stayed subscribed after losing the race, receiving the winning driver's
   live GPS and the rider's pickup/drop for the whole ride.

3. **Vehicle-type filtering happened in Python after Redis truncation.**
   One global `drivers:geo` key, `GEOSEARCH ... COUNT 50`, then filter. In
   a bike-dense area a sedan request matched zero drivers.

4. **Commission came from a global env var defaulting to `"0"`.** The
   per-zone `RateCard.commission_percent` was documented as
   "informational" and never used in the money path.

5. **Rider credits and driver settlement were the same `Wallet` row**, so a
   driver who also rides could spend settlement money on rides — and the
   closed-loop credit posture of [ADR-0003](0003-closed-loop-wallet.md)
   was unenforceable, because settlement money is not a credit.

6. **Ghost drivers.** Geo entries were removed only on a clean WebSocket
   disconnect. A killed app or a crashed worker left a driver matchable
   indefinitely.

7. **`/ride/trip/<id>/details/` had no authorisation check** on either the
   cache or the DB path. Driver name, phone number, rating and number plate
   were readable by walking trip ids — which defeats the "driver's phone is
   never displayed" posture entirely.

8. **The mobile app held the Google Maps key**, called Places and Geocoding
   directly, relayed web traffic through the public `corsproxy.io`, and
   routed via the public OSRM demo server.

## Decision

### 1. The trip transaction is DB-only

Everything inside `transaction.atomic()` in the trip state machine is now a
database write. Push notifications, Redis cache writes, receipt issuance
and gateway calls are registered with `transaction.on_commit`. Receipt
issuance moved to a Celery task (`ride.issue_receipt_for_trip`).

The Cashfree order is no longer created at completion at all — completion
records payment *intent*, and `POST /payments/create-order/` (which the app
already calls from the payment screen) creates the order.

**Rule going forward:** if it can fail slowly or fail remotely, it does not
belong inside a transaction that holds a lock on a hot row.

### 2. Trip-group membership follows assignment

`TripStatusConsumer` classifies a connection as `rider`,
`assigned_driver`, `candidate_driver` or `none`. Only the first two join
the `trip_<id>` channel group. A candidate driver may send
`{"action": "accept"}` and is added to the group only after their accept
transaction commits.

Losing drivers get an explicit `trip_taken` event — over their own
`driver_<id>` group, from a Redis offer set recorded at fanout time — and
their trip socket is closed with code 4005.

### 3. Dispatch is a rolling fanout over per-vehicle-type indices

- One geo key per vehicle type (`drivers:geo:sedan`, `drivers:geo:bike`).
  Filtering happens in the Redis query, never after truncation.
- Offers go out in expanding waves (default 1500m → 3000m → 5000m, 20s
  apart), skipping drivers already offered, and stop as soon as the trip
  leaves `requested`.
- Accept timeout drops from 600s to 90s.
- Presence heartbeats (`driver:heartbeat:<id>`, TTL 45s) are refreshed on
  every location ping; a Celery beat job sweeps geo entries whose heartbeat
  has lapsed every 60s, and `nearby_drivers` additionally filters them at
  read time.
- Vehicle type is cached in Redis, so the location hot path no longer does
  a Postgres read per ping.

### 4. Commission is priced by zone

`Trip.zone` is stamped at booking. `pricing.services.commission_percent_for_trip`
resolves `RateCard.commission_percent` for the trip's zone and vehicle type
**as of the time the trip was requested**, so a rate change never re-prices
history. `settings.PLATFORM_COMMISSION_PERCENT` remains only as a
last-resort fallback, and is now a `Decimal` defaulting to 18.

### 5. Wallets are scoped by role

`Wallet` gains a `scope` field (`rider` | `driver`) with a unique
constraint on `(user, scope)`. `get_wallet(user, scope)` is the only
supported accessor. Migration `rider/0011` re-scopes every existing wallet
belonging to a user with a Driver profile to `driver`.

### 6. Cancellations are attributed

`Trip.cancelled_by`, `cancellation_reason` and `cancellation_fee` are set
by all three cancel paths (rider REST, driver REST, WebSocket) and by the
auto-cancel task. The `status_code` vocabulary stays a single `cancelled`
value so the four clients are unaffected — attribution is data, not a new
state.

### 7. Maps is a server capability, not a client one

`/ride/maps/geocode`, `/ride/maps/place-details`, `/ride/maps/reverse-geocode`
and `/ride/maps/directions` are the only ways the app touches Google. The
key stays server-side; the proxy rate-limits per user and caches.

## Consequences

**Good**

- Trip completion is a short DB transaction; SES or Cashfree being slow no
  longer stalls a ride ending.
- A ride offered to twelve drivers clears eleven screens the instant it is
  taken.
- A sedan rider in a bike-heavy area gets matched.
- The platform actually earns its commission, per city.
- Settlement money cannot be spent as rider credit.
- The Google Maps key can be rotated and restricted to our server IPs; the
  key currently in the app bundle must be treated as public and rotated.

**Costs / follow-ups**

- The rolling fanout takes up to ~40s to reach the widest radius. In a
  thin-supply city that is slower to first-offer than the old blast. Tune
  `DISPATCH_RADIUS_WAVES_M` / `DISPATCH_WAVE_SECONDS` per city once there
  is real accept-rate data.
- `nearby_drivers` with no vehicle type now scans all type keys. Fine at
  Phase-0 scale; revisit if the ops live map gets heavy.
- `Wallet` is now a FK, not a OneToOne. Any new code must pass a scope.
- Receipts are eventually consistent — a rider who opens the receipt screen
  within a second of completion may need one refresh.

## Not done in this batch

- Two-factor for admin login, and moving ops-console tokens out of
  `localStorage` (tracked in [ADR-0005](0005-ops-web-console.md)).
- The driver app, which still does not exist and remains the single largest
  gap between this platform and an operating business.
- Multi-country groundwork (currency on money rows, tax abstraction,
  per-country payment routing). Explicitly deferred: Phase-0 is India-only.

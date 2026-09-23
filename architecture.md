# SaaradhiGo architecture

- **Date:** 2026-09-23
- **Scope:** what exists now, and the shape it is converging on.
- **Principle:** boring architecture. One Django process family, PostgreSQL for
  truth, Redis for live state, Celery for durable async, S3 for artifacts. No
  microservices, no Kafka, no Kubernetes.

## Current

```
        Rider app            Driver app           Ops console / web
        (Flutter)            (Flutter)            (Django templates + Next.js)
            │                     │                        │
            ├───── HTTPS ─────────┼──── HTTPS ─────────────┤
            └───── WebSocket ─────┘                        │
                        │                                  │
                        ▼                                  ▼
        ┌──────────────────────────────────────────────────────────────┐
        │  backend  (Daphne / ASGI, one Railway service)               │
        │                                                              │
        │  HTTP adapters              WebSocket adapters               │
        │   servers/*/views.py         servers/consumers.py            │
        │   DRF, JWT                   RideRequestConsumer             │
        │                              TripStatusConsumer              │
        │                              DriverLocationConsumer          │
        │                              AdminDashboardConsumer          │
        │                              TripChatConsumer                │
        │                                                              │
        │  Domain services (the part that is growing)                  │
        │   pricing/services.py      quote_fare  <-- single fare engine │
        │   pricing/fare_shadow.py   observe-only comparison           │
        │   ride/dispatch.py         wave orchestration                │
        │   ride/location_trail.py   GPS sampling + durable writer     │
        │   ride/actual_metrics.py   measured distance / duration      │
        │   ride/promos.py           promo rules (NOT wired to booking) │
        │   driver/earnings.py       read model over the ledger        │
        │   driver/services.py       withdrawals + payouts             │
        │                                                              │
        │  STILL INSIDE consumers.py, not yet extracted:               │
        │   _create_trip  _accept_trip  _update_trip_status            │
        │   _create_payment_on_complete  _process_refund_on_cancel     │
        └───┬──────────────┬───────────────┬──────────────┬────────────┘
            │              │               │              │
            ▼              ▼               ▼              ▼
     ┌────────────┐  ┌───────────┐  ┌────────────┐  ┌──────────────┐
     │ PostgreSQL │  │   Redis   │  │   Celery   │  │  S3-compatible│
     │  durable   │  │ ephemeral │  │   async    │  │   storage     │
     │   truth    │  │   state   │  │            │  │               │
     └────────────┘  └───────────┘  └─────┬──────┘  └──────────────┘
                                          │
                                   ┌──────┴───────┐
                                   │ celery       │
                                   │ worker + -B  │  <-- Beat still embedded
                                   └──────────────┘
                                          │
                                          ▼
                              ┌────────────────────────┐
                              │ External providers     │
                              │  Cashfree (pay/payout) │
                              │  Firebase FCM (push)   │
                              │  AWS SNS (OTP SMS)     │
                              └────────────────────────┘
```

## Target

Two differences only. Both are in flight, neither is speculative.

```
     ... adapters unchanged ...
            │
            ▼
     Domain services            <-- _accept_trip, lifecycle and completion
     ride/services/*.py             extracted out of consumers.py, one at a time
            │
            ├──► PostgreSQL ── durable truth
            ├──► Redis ─────── live/ephemeral only
            ├──► Celery ────── worker x N   +   beat x 1   <-- separated
            └──► S3 ────────── KYC + receipts
```

## Authoritative ownership

Every row of this table is a decision, not an observation: if two stores could hold
the same fact, the table says which one is right.

### PostgreSQL — durable business truth

| Owns | Tables |
|---|---|
| Trips and lifecycle | `ride_trip` (+ `TripStatus`), `cancellation_*` |
| Fare snapshots | `ride_farepricing` (quote today; quote + final once the snapshot columns land) |
| GPS history | `ride_triplocationpoint` |
| Money ledger | `rider_wallettransaction` — **authoritative for balances** |
| Transaction display log | `payments_transactionhistory` — *not* authoritative |
| Payments | `payments_payment` |
| Withdrawals / payouts | `driver_withdrawalrequest` |
| Settlement economics | `trip_settlement` — **proposed, not implemented** |
| KYC metadata | `driver_driver`, `driver_vehicle` (document *keys*, not bytes) |
| Pricing configuration | `pricing_servicezone`, `pricing_ratecard` (pricing now immutable once created), `pricing_platformsettings` |
| Promos | `ride_promocode`, `ride_promoredemption` (redemption **not wired**) |
| Receipts | `ride_receipt`, versioned; the GST document of record |
| Fare shadow | `trip_fare_shadow` — observational, read by no money path |
| Admin audit | `admin_audit_adminauditlog` |

### Redis — live and ephemeral only

Nothing here is a source of truth. All of it can be lost without losing business
state; losing it degrades liveness, not correctness.

| db | Contents |
|---|---|
| 0 | Celery broker |
| 1 | Django cache — including the OTP cache |
| 2 | driver GEO index, heartbeats, dispatch offer generations, active-trip cache |
| 3 | `driver_location_stream` — the GPS event stream |
| 4 | Channels layer (WebSocket groups) |

The rule that makes this safe: **the trail writer resolves a ping's trip from
PostgreSQL, never from Redis**, so a stale Redis view cannot cause a wrong
attribution.

### Celery — durable async execution

| Beat schedules (true periodic) | Cadence |
|---|---|
| `ride.persist_location_trail` (GPS drain) | 60 s |
| `pricing.fare_shadow_sweep` | 15 min |
| `payments.reconcile_stuck_payments` | 5 min |
| `payments.reconcile_stuck_withdrawals` | 10 min |
| `driver.block_expired_driver_licenses` | daily 02:00 |
| `driver.sweep_stale_driver_presence` | periodic |

| NOT Beat — broker/worker ETA or on-commit | Trigger |
|---|---|
| `servers.ride.tasks.auto_cancel_trip` | countdown at booking |
| `ride.dispatch_wave` | countdown per wave |
| `ride.compute_trip_actuals` | countdown after completion |
| `ride.issue_receipt_for_trip` | `transaction.on_commit` |
| `auth_user.send_push_notification_task` | fired inline |

**This distinction matters operationally.** Stopping Beat stops the six scheduled
sweeps; it does **not** stop auto-cancel, dispatch waves, actuals or receipts,
because those are queued by the web process with an ETA and executed by the worker.
A Beat outage is therefore a reconciliation outage, not a ride outage — and anyone
verifying a Beat migration must check both lists separately.

### S3-compatible storage

| Prefix | Contents | Access |
|---|---|---|
| `license_docs/`, `license_docs_back/` | driver licences | private, presigned |
| `rc_docs/`, `permit_docs/`, `insurance_docs/`, `fitness_docs/`, `puc_docs/` | vehicle compliance | private, presigned |
| `vehicle_pic/` | vehicle photos | public-read media |
| receipts | generated PDFs | private, presigned |

QA runs on a Railway bucket with its own scoped credentials; production has **no
storage configured yet**.

### External providers

| Provider | Used for | State |
|---|---|---|
| Cashfree payments | rider order creation, webhooks, refunds | live |
| Cashfree payouts | driver UPI transfers | live, **contract unverified** |
| Firebase FCM | push | live |
| AWS SNS | OTP SMS | live (bypassed for QA test phones) |

## The financial chain

Each arrow is a transition that must be auditable, idempotent where required,
testable, and explainable. Current state in brackets.

```
Trip                          [PostgreSQL, lifecycle guarded by a state machine
                               under SELECT FOR UPDATE]
  ↓
GPS evidence                  [PROVEN in QA: Redis stream -> durable
                               TripLocationPoint, idempotent on
                               (trip, source_event_id)]
  ↓
Actual metrics                [PROVEN in QA: distance from the trail, duration from
                               OTP-gated lifecycle timestamps, with coverage_ratio,
                               max_gap and rejected_segments]
  ↓
Fare snapshot                 [PARTIAL: FarePricing holds the quote; the columns that
                               make it explainable without today's RateCard are
                               written and tested but NOT merged]
  ↓
Final fare                    [NOT ENABLED. final_fare is NULL on every trip and no
                               code writes it]
  ↓
Payment                       [live; reads final_fare or estimated_fare]
  ↓
WalletTransaction             [authoritative ledger; idempotent on
                               TRIP_<id>_EARNING]
  ↓
TripSettlement                [PROPOSED, not implemented — the economics behind the
                               ledger movement]
  ↓
Driver earnings / receipt / reporting
                              [earnings read model fixed and live; receipts live and
                               versioned; admin revenue still recomputes commission
                               at render time until TripSettlement lands]
```

## What is deliberately NOT here

No message bus, no service mesh, no separate read store, no CQRS, no container
orchestrator. The problems this platform has are ordering, idempotency and
provenance inside one process family — none of which a network boundary fixes, and
all of which it makes harder to test.

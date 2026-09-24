# SaaradhiGo Extended Pilot Readiness Report

- **Date:** 2026-09-24
- **Scope:** extended autonomous engineering pass following the resolved
  sustained-GPS lifecycle blocker.
- **PII:** no coordinates, phone numbers, OTPs, JWTs, document URLs or passwords
  appear below.

## Executive summary

The platform stopped relying on hope in three places where it previously did.

**A command is now acknowledged, not assumed.** Before this run, the only success
signal for every lifecycle command was a `group_send` broadcast — and
`channels_redis` silently drops messages to a channel over capacity. A committed
completion could therefore go unacknowledged *by design*, with no transport failure
involved at all, while the driver app sat waiting forever. That is the same failure
class as the resolved blocker, reached by a different route. There is now a direct,
post-commit, correlated `command_ack`, and the client waits for it, retries the same
`command_id`, and treats `already_done` as the success it is. Demonstrated with a
negative control: 125 messages sent through a group, 100 delivered.

**A booking can be retried.** A double-tap or a reconnect used to create two trips —
two dispatch chains competing for the same drivers, two surge-demand records, two
auto-cancel deadlines. A per-rider partial unique index in PostgreSQL now makes a
retry return the original trip and perform none of the side effects of booking,
proven under genuine concurrency rather than argued.

**Two bounds that were documented but not enforced now are.** The per-trip GPS point
cap was checked against the wrong thing and could never trigger (5400 events stored
5400 rows). SOS had no de-duplication, so one panic button paged ops repeatedly.

Alongside those: GPS ingestion failure and ledger-duplicate suppression are no longer
silent; raw coordinates no longer reach the SOS log; raw GPS trails are pinned
unreachable; and `pip-audit` went from 56 vulnerabilities across 9 packages to 16
across 3 — with the remaining 15 traced to a single root cause rather than left as a
backlog.

## Current HEAD

| Repo | Branch | Commit |
|---|---|---|
| SaaradhiGo-backend | `dev` | **`b3444d7`** |
| SaaradhiGo-driver | `feat/driver-app-rebuild` | `ca6fbe9` |
| SaaradhiGo-docs | `fix/promo-fare-integrity` | (this document) |

All working trees clean. Backend `dev` deployed to QA throughout.

## Merged

| Branch | Merge | Purpose | Tests | QA evidence |
|---|---|---|---|---|
| `feat/command-ack-protocol` | `74185dd` | direct post-commit `command_ack` with committed/already_done/rejected | 16, incl. a negative control demonstrating `group_send` dropping 25 of 125 | 16/16 checks on deployed QA |
| `feat/trip-request-idempotency` | `494303a` | `client_request_id` + partial unique index, per rider | 9, incl. real-PostgreSQL concurrency and a DB-level constraint test | booking retry returns the same trip |
| `test/reconnect-state-machine` | `c6ecde1` | reconnect into durable truth in every state | 17, incl. a deliberately corrupted Redis cache | reconnect reports `completed` |
| `feat/gps-ingest-observability` | `facd58f` | GPS ingest failure + ledger duplicates made visible | 5 | — |
| `test/long-ride-soak` + cap fix | `9a5631a` | soak to 4 simulated hours; enforced the per-trip GPS cap | 9 | — |
| `test/gps-access-control` | `e12eb25` | pin that raw trails stay unreachable | 10 | — |
| `fix/sos-reliability` | `93d8b5a` | stop logging SOS coordinates; collapse repeat presses | 9 | — |
| `chore/dependency-security-safe-subset` | `01a9d47` | 6 packages upgraded, 40 advisories closed | 598 + 60 on real infra | — |
| `ci/postgres-suite-per-file` | `b3444d7` | make the postgres gate reliable | 88/88 | — |
| driver `feat/command-ack-retry` | `ca6fbe9` | client waits for the ack, retries safely; **CI added** | 74 on GitHub Actions | `flutter analyze`: no issues |

## Ready but unmerged

| Branch | Why not merged |
|---|---|
| `feat/fare-snapshot-schema` | Not reached this run. Needs the schema review against QA evidence described in Priority 8 before it earns a merge; it is additive but it is fare provenance, so it deserves that review rather than a rebase-and-ship. |
| `ops/celery-production-readiness` | Not reached. Needs the task-by-task durability audit (idempotent? financial? safe to redeliver?) rather than a global `acks_late`. |
| `cashfree-pg` upgrade | Deliberately not attempted. See Payouts. |

## Command reliability

The protocol is additive; `trip_status_update` and `error` are unchanged and still
sent, `command_id` is optional, and the server never invents one.

```
client → {"action":"complete","command_id":"<uuid>"}
server → {"type":"command_ack","command":"complete","command_id":"<same>",
          "status":"committed|already_done|rejected",
          "trip_status":"completed","reason":"<code>"}
```

`committed` means the transaction returned — asserted by showing that an independent
database connection can see the row the moment the ack lands. `already_done` is the
retry case and the reason this exists: before it, a retry after a lost ack got
"Invalid status transition: cannot change from completed to completed", correct in
that nothing ran twice and useless to a client that could not tell it from a real
failure.

**No command-ID table was added, and none is needed.** For a lifecycle transition the
trip's own status *is* the idempotency record, because the target state is the dedup
key. `command_id` is for correlation only. Trip creation is the case that genuinely
needs a durable key, since the row does not exist yet — which is why that is a
separate mechanism.

Client side (`CommandDispatcher`): waits for the correlated ack, retries the same
`command_id` with bounded exponential backoff and jitter, never retries a
`rejected`, and on an unacknowledged command reports `unknown` and re-reads the trip
from the server rather than claiming either outcome. It also accepts the legacy
`trip_status_update` / `error` / `cash_payment_confirmed` frames, so a version skew
or a rollback cannot leave every command hanging — the pre-existing lifecycle tests
answer with *only* the old broadcast and pass unchanged, which is the compatibility
proof.

## Long-ride reliability

Accelerated by frame count at the real 2.5-second sampling rate, because sleeping
would test the clock.

| Simulated ride | Frames | Ingested | `complete` committed | Command-socket buffer |
|---|---|---|---|---|
| 30 min | 720 | 720 (100%) | 0.22s | 0 |
| 60 min | 1440 | 1440 (100%) | 0.09s | 0 |
| 120 min | 2880 | 2880 (100%) | 0.16s | 0 |
| 240 min | 5760 | 5760 (100%) | 0.09s | 0 |

Completion latency does not scale with ride length (5 min 0.14s vs 60 min 0.14s).

**Wall-clock QA:** a 20-minute real ride with the driver's command socket undrained
and its keepalive off completed, with all 120 positions delivered to the rider and
`final_fare` NULL. The harness initially reported this as a failure — its own
15-minute access token expired mid-ride, so the verification read returned 401. A
re-read with a fresh token confirmed `completed`. Harness defect, now fixed, and
worth knowing: **a normal long ride outlives the 15-minute access token.** The apps
handle it (the driver client refreshes on 401 and replays, collapsing concurrent
refreshes); the WebSocket authenticates at connect and correctly survives.

## WebSocket / backpressure

Audited every high-frequency path. After the two fixes that resolved the blocker, the
paths themselves came out healthy, and that is recorded so it is not re-litigated:

- dispatch fan-out already uses per-driver `group_send` (drops rather than blocks),
  isolates per-driver exceptions, and enqueues FCM as Celery tasks from a single bulk
  query;
- completion side effects are all `transaction.on_commit`, so no push, receipt or
  gateway call happens inside the row lock;
- chat has authentication, a membership gate and a 2000-character cap;
- the broad `except Exception` handlers in `redis_client` are cleanup and
  observability helpers returning a falsy default, not data paths.

The gap was **signal, not structure**. A location frame that failed to reach Redis
produced one warning among hundreds and no aggregate, so "why did this ride's GPS
trail stop?" had no answer. `DriverLocationConsumer` now warns once per connection on
the first refusal and emits one `gps_session_summary` at disconnect — WARNING when
frames were lost so it survives production's log level, INFO when none were, silent
for an idle connection, and never a coordinate.

## Trip idempotency

`Trip.client_request_id`, nullable, with a **partial unique index scoped per rider**
(migration 0013, purely additive). Scoped per rider because client ids are not
coordinated across devices and a global constraint would let one rider's key block
another's — asserted.

The lookup happens *before* the fare is computed, because
`estimate_amount(record_demand=True)` registers demand for surge; a double-tap that
reached pricing would push the rider's own surge multiplier up before quoting them.

PostgreSQL is the arbiter rather than Redis, and the concurrency test is written to
distinguish them: two simultaneous identical bookings both find nothing, both insert,
the index refuses one, and the loser re-reads the winner's row. One trip, both
sockets answered.

Side effects measured rather than inferred — the `ride_requests` stream went 97 → 98
on the booking and stayed at 98 on the retry. A negative control asserts that a
client sending no key still creates two trips, so compatibility is pinned.

## GPS

**Durability.** Ingestion is 100% at every ride length tested.

**Growth is bounded by duration, not transmit rate** — the number retention planning
needs, measured by writing realistically spaced stream entries (`recorded_at` derives
from the Redis stream ID, so this is faithful rather than simulated):

| Ride | Durable points | Per minute |
|---|---|---|
| 10 min | 120 | 12.0 |
| 30 min | 360 | 12.0 |
| 60 min | 720 | 12.0 |

Exactly 12 per minute regardless of how fast the client transmits. Indicative
storage: ~720 rows/hour/ride, so a 20-minute average ride is ~240 rows. These are
measurements of the sampler, not a production forecast — rides/day is a business
input nobody has given me, so no 30/90/365-day totals are asserted here.

**The per-trip cap was not enforced.** `MAX_POINTS_PER_TRIP = 5000` is documented as
defence against a broken client; the check lived inside `sample()`, compared against
the length of the current batch, and `sample()` is not on the drain path at all. With
batches of 500 it could never trigger. Demonstrated: 5400 eligible events stored all
5400. Now 5000 exactly, with the first breach logged.

**Access control.** Raw trails are reachable by nothing — no serializer, no view, no
route, no admin registration. That is pinned by tests that fail the moment a read
path appears, including a serializer check that walks every serializers module by
import. A rider reading their own completed trip gets no trail, because nobody needs
raw location history to see a receipt. `admin_live_locations` is the one endpoint
returning coordinates; it serves live Redis positions, not history, and is gated on
`IsAuthenticated + IsAdmin`.

**Retention itself is not implemented** — no deletion mechanism was built, and none
should be until someone decides the dispute/insurance window.

## Fare snapshot

Not advanced this run. `feat/fare-snapshot-schema` remains unmerged and unreviewed
against the QA evidence. `final_fare` is NULL on every trip and nothing written this
run touches it — asserted in the ack suite, the idempotency suite and the QA script.

## RateCard

Already complete before this run and re-verified: 25 tests pass. Pricing fields are
refused on an existing card through both `save()` and queryset `update()`, lifecycle
fields stay editable, `new_version()` is the supported price change, and an audited
correction path exists. A historical quote stays explainable after a price change.

## Settlement

Not implemented. `TripSettlement` remains proposed. No second ledger was created;
`WalletTransaction` remains authoritative. What did change is that its idempotency
guard is no longer silent: `credit_driver_wallet`'s `except IntegrityError: return`
is what stops a driver being credited twice, and a rising rate there now shows up as
`driver_earning_duplicate_suppressed` rather than nothing.

Repeated completion is proven financially idempotent: six completions leave the money
snapshot byte-identical — one Payment, one TransactionHistory, no wallet row, no
receipt, `completed_at` unchanged, `final_fare` NULL. One honest limitation carried
forward: these are unpaid cash trips, so no settlement or receipt ran at all, and
"settlement unchanged" is proven only in the trivial sense.

## Earnings / Promo / Payments

Untouched this run. Promo remains inactive, tax unchanged, no rider payable changed.

## Payouts

The remaining Cashfree risk is unchanged and unverified. The single new finding is a
dependency one: **`cashfree-pg 3.2.12` hard-pins `urllib3<2.1.0` and
`sentry-sdk<1.33.0`**, which makes all 15 remaining advisories unreachable. The SDK
is three major versions behind (6.0.1 current). Upgrading it changes provider
interaction semantics, so it is a human decision requiring sandbox verification, not
an autonomous merge.

## Celery

Not advanced. `ops/celery-production-readiness` remains unmerged and the task-by-task
durability audit is not done. Beat still runs embedded in the worker (`-B`).

## SOS

Two real findings, both fixed.

**Raw coordinates in the log.** `SOS RAISED ... lat=... lng=...` put a person's exact
position into plaintext logs at WARNING, where it is retained, shipped and
searchable. SOS is precisely where location is most sensitive. The line now carries
the event id, role, trip and `has_location`; the coordinates stay in the database.

**No de-duplication.** Five presses of one panic button created five events and five
fan-outs — how a real alert gets tuned out. Now one event and four visible
`sos_repeat_collapsed` lines, so a stuck button is still diagnosable.

Every dedupe rule fails **open**, because suppressing a real emergency is far worse
than paging twice: a repeat collapses only when user, trip and event type all match,
the original is still `open`, and it is inside 120 seconds. Panic-then-medical is an
escalation; an SOS after ops acknowledged is new; a repeat minutes later is new; and
the driver raising SOS where the rider already did is never collapsed — two different
people in danger. Decided in PostgreSQL, so a cache miss cannot cause a duplicate and
a cache hit cannot cause a drop.

**Under load:** SOS behind 500 queued GPS frames answered in 0.05s (HTTP 201) against
a 0.36s quiet baseline, both events durable.

## Receipts / Redis failure behaviour

Not audited this run.

## PostgreSQL

Migration 0013 is additive (one nullable column, one partial unique index) and
applied cleanly in QA. No pending migrations. The per-trip GPS count uses a single
`GROUP BY` per drain rather than a query per trip, so it does not scale with the
number of active rides.

## Security

**Dependencies: 56 → 16 vulnerabilities, 9 → 3 packages.** Upgraded with verification
matched to what each could break — the full 598-test suite for PyJWT and cryptography,
and 60 WebSocket/lifecycle tests on real infrastructure for daphne and Twisted, since
those two *are* the transport.

| Package | From → To | Advisories |
|---|---|---|
| cryptography | 44.0.1 → 50.0.1 | 5 |
| setuptools | 65.5.0 → 84.0.0 | 4 |
| PyJWT | 2.10.1 → 2.15.0 | 3 |
| pyOpenSSL | 25.1.0 → 26.4.0 | 2 |
| daphne | 4.1.2 → 4.2.3 | 1 |
| Twisted | 25.5.0 → 26.4.0 | 1 |

Remaining: urllib3 (7) and sentry-sdk (1) blocked by the Cashfree pin; pytest 8 → 9 is
a dev-only major bump left for its own change.

**WebSocket:** the trip-socket participation gate was re-verified now that the
greeting carries trip state — an unassigned driver and an unrelated rider are both
refused 4003 rather than shown another rider's trip status. The `command_id` is
length-bounded so it cannot be used to push arbitrary payload back out. A full
consumer-by-consumer IDOR/rate-limit audit (Priorities 29–30) was **not** completed.

## CI

The backend already had lint, SQLite tests, postgres-concurrency and EC2 deploy. One
change: the postgres job now runs **one process per file**. Running all 88 in a single
process is the one configuration in which they fail — see New defects. Per-file is
88/88, and that is how the suite was verified throughout.

**The driver app had no CI at all.** It now runs `flutter analyze` and `flutter test`
on every push: no analyzer issues, 74 tests passing. That also converted the app work
from unverifiable in this environment (no Dart SDK available) into verified.

## Deployment

Railway is the QA path and works. The EC2 deploy job in `deploy.yml` is untouched:
production ownership is still undetermined, and per the brief that means document and
leave alone rather than remove deployment capability.

## Production readiness

Migration behaviour, storage, Celery topology and secrets are unchanged from the
previous report. Nothing in this run requires a production change, and no production
infrastructure was touched.

## Architecture

```
   Rider app            Driver app                 Ops console
   (Flutter)            (Flutter)                  (Django + Next.js)
       │                     │                          │
       │  HTTPS + WebSocket  │                          │
       └──────────┬──────────┘                          │
                  ▼                                     ▼
   ┌───────────────────────────────────────────────────────────────┐
   │  backend  (Daphne 4.2.3 / ASGI, one Railway service)          │
   │                                                               │
   │  WebSocket adapters                                           │
   │   DriverLocationConsumer   ─┐                                 │
   │   TripStatusConsumer        ├─ LocationBroadcastMixin:         │
   │   RideRequestConsumer       │   location frames COALESCE onto  │
   │   AdminDashboardConsumer   ─┘   a depth-1 queue drained by a   │
   │                                 task; commands keep a direct,  │
   │                                 ordered, blocking send         │
   │                                                               │
   │  command_ack  ── direct, post-commit, correlated              │
   │  connection_established.trip_status ── durable, from PostgreSQL│
   │                                                               │
   │  Domain services                                              │
   │   pricing/services.py    quote_fare (single fare engine)      │
   │   ride/dispatch.py       waves, per-driver group_send          │
   │   ride/location_trail.py sampling + per-trip cap (enforced)    │
   │   ride/actual_metrics.py measured distance / duration          │
   │   sos/views.py           durable first, fan-out on_commit      │
   └───┬──────────────┬───────────────┬──────────────┬─────────────┘
       ▼              ▼               ▼              ▼
  PostgreSQL      Redis           Celery        S3-compatible
  durable         live only       async         KYC + receipts
  TRUTH           (never          worker + -B
                   authoritative)  (Beat still embedded)
```

Unchanged in shape: a modular monolith. No microservices, no Kafka, no Kubernetes.

## Pilot readiness matrix

| Area | | One line |
|---|---|---|
| Rider auth | GREEN | OTP + JWT working; 15-minute access token is shorter than a long ride and the apps refresh correctly. |
| Driver auth | GREEN | Same, plus verified 401-refresh-and-replay in the driver client. |
| KYC | GREEN | Real audited approval path; self-approval boundary hardened earlier. |
| Ride creation | GREEN | Now idempotent on a per-rider key with a PostgreSQL constraint. |
| Dispatch | GREEN | Celery-owned, per-driver fan-out that drops rather than blocks, FCM bulk-queried. |
| Acceptance | GREEN | Race resolved under `select_for_update`; losing candidates dismissed. |
| Lifecycle commands | GREEN | Acknowledged post-commit, correlated, and safely retryable. |
| Reconnect | GREEN | Greeting carries durable status from PostgreSQL; proven against a corrupted cache. |
| GPS | GREEN | 100% ingestion to 4 simulated hours; growth bounded at 12 points/minute; per-trip cap now enforced. |
| Actual metrics | GREEN | Computed post-completion, observe-only. |
| Fare shadow | GREEN | Observational; reads no money path. |
| Final fare | GREEN (as intended) | NULL everywhere; nothing writes it. |
| Promo | GREEN (as intended) | Inactive and unwired. |
| Payments | YELLOW | Live and idempotent on completion, but a settled-trip rehearsal has never run. |
| Settlement | RED | `TripSettlement` still proposed; economics recomputed at render time. |
| Earnings | YELLOW | Read model correct; duplicate suppression now visible; still no settlement source. |
| Payouts | RED | Cashfree contract unverified, and its SDK pin blocks 15 security advisories. |
| Receipts | YELLOW | Live and versioned; retry/S3-failure behaviour not audited this run. |
| SOS | GREEN | Durable-first, deduped without ever failing closed, unaffected by 500 queued frames. |
| PostgreSQL | GREEN | Additive migration applied; no N+1 introduced; invariants under row locks. |
| Redis | YELLOW | Correctly non-authoritative, but degradation behaviour is untested and a cross-test anomaly is unexplained. |
| Celery worker | YELLOW | Working; per-task durability audit not done. |
| Celery Beat | YELLOW | Still embedded via `-B`; separation prepared but not executed. |
| S3 | GREEN (QA) | QA bucket scoped; production storage not created. |
| CI | GREEN | Backend lint/tests/postgres-per-file; driver app CI added from nothing. |
| Deployment | YELLOW | Railway QA proven; production owner still undetermined. |
| Migrations | GREEN | Additive, applied, failure-blocks-rollout proven earlier. |
| Security | YELLOW | 40 advisories closed; 15 blocked by one SDK pin; full WS IDOR audit outstanding. |
| Observability | GREEN | Lost command sockets, GPS ingest failure, coalescing, ledger duplicates and SOS repeats all emit structured, PII-free events. |

## New defects discovered

| Defect | Severity | Impact | Status |
|---|---|---|---|
| Lifecycle success signal travelled only on a droppable `group_send` | **High** | A committed completion could go unacknowledged with no failure anywhere; the app waits forever | **Fixed** (`74185dd`), negative control included |
| `MAX_POINTS_PER_TRIP` never enforced | Medium | A broken or hostile client could write unbounded GPS rows for one trip | **Fixed** (`9a5631a`); 5400 → 5000 |
| SOS logged raw coordinates | Medium (privacy) | A person's exact position in retained, shipped, searchable logs | **Fixed** (`93d8b5a`) |
| SOS had no de-duplication | Medium (safety-adjacent) | One panic button paged ops repeatedly, training them to ignore it | **Fixed** (`93d8b5a`), failing open by design |
| Ride booking not idempotent | Medium | Double-tap created two trips, two dispatch chains, doubled surge demand | **Fixed** (`494303a`) |
| GPS ingestion failure had no aggregate signal | Medium | "Why did the trail stop?" was unanswerable | **Fixed** (`facd58f`) |
| Ledger duplicate suppression was silent | Low | The guard protecting against double-crediting a driver emitted nothing | **Fixed** (`facd58f`) |
| Postgres suite fails 8/88 in a single process | Low (test infra) | A `SET` succeeds and an immediate `GET` returns None, no eviction, no error | **Contained**, not root-caused — CI runs per-file; 88/88 |
| Driver app had no CI | Low | Every app change unverified until someone remembered | **Fixed** (driver `ca6fbe9`) |

## Remaining pilot blockers

Two, and neither is technical debt:

1. **Payout correctness depends on an unverified Cashfree contract.** The refund-on-
   ambiguous-failure path can plausibly pay a driver *and* refund the platform. No
   amount of internal testing settles this; it needs sandbox evidence.
2. **Settlement economics are recomputed at render time.** Until `TripSettlement`
   exists, "what did this ride actually earn" is answered by today's RateCard rather
   than by recorded fact. That is survivable for a small pilot and not for a dispute.

Everything else above is ordinary work, not a blocker.

## Human actions required

1. **Decide on the Cashfree SDK.** `cashfree-pg 3.2.12 → 6.0.1` would unblock 15
   security advisories that are otherwise unreachable. It changes payment-provider
   interaction, so it needs: a sandbox run against the real contract, confirmation
   that order creation / webhook / refund shapes are unchanged or adapted, and your
   authorisation. Nothing else closes those advisories.
2. **Run the Cashfree payout sandbox checklist** (already written) and record the
   duplicate-transfer behaviour. This is the gating evidence for payout safety.
3. **Name the production deployment owner** — Railway or EC2. The EC2 workflow is
   deliberately untouched until that is decided.
4. **Merge the driver app branch to its integration branch** if `feat/driver-app-rebuild`
   is not itself the target; CI is green on it (74 tests, no analyzer issues).
5. **Confirm the GPS retention window** (dispute / insurance / analytics). The
   measured growth is 12 points/minute/ride; the policy decision is yours and no
   deletion mechanism should be built before it.

## Next five actions

1. Rebase and review `feat/fare-snapshot-schema` against the QA ride evidence, drop
   unused fields, and merge if it stays additive — it is the prerequisite for
   settlement.
2. Implement `TripSettlement` as immutable economic evidence written inside the
   existing financial transaction, with database uniqueness and no second ledger.
3. Audit `ops/celery-production-readiness` task by task (idempotent? financial? safe
   to redeliver?) rather than enabling `acks_late` globally, then separate Beat.
4. Root-cause the single-process postgres anomaly, or prove it is purely a local
   Redis artifact, and collapse the CI loop back to one invocation.
5. Complete the consumer-by-consumer WebSocket IDOR and rate-limit audit
   (Priorities 29–30), keeping telemetry and lifecycle quotas separate so SOS can
   never be throttled by a GPS budget.

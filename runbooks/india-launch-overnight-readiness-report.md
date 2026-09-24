# SaaradhiGo India Launch Readiness

- **Date:** 2026-09-24
- **Run type:** overnight autonomous engineering and product-readiness pass.
- **PII:** no phone numbers, OTPs, JWTs, coordinates, document URLs or secrets appear
  below.

## Executive summary

The financial core is now built. SaaradhiGo can explain a fare and explain a
settlement from recorded facts instead of recomputing them, and for the first time a
QA ride was driven all the way through cash confirmation — trip 40, fare 124.40,
commission 24.88, driver net 99.52, `final_fare` still NULL. Every previous rehearsal
stopped at completion on an unpaid trip, so `credit_driver_wallet` had never actually
run end to end.

Two immutable records landed: `FarePricing` gained the columns that let a historical
fare be explained without today's rate card, and `TripSettlement` records the
commission rate **as applied** so editing a rate card cannot rewrite last month's
revenue. A test supersedes every live rate card with a 40% commission and asserts the
settled row does not move. Four threads settling one trip concurrently produce one
settlement, one ledger row and a balance of exactly 492.00 on a 600.00 fare.

The most important non-financial finding was in the apps, not the backend. **Both
mobile apps defaulted to the QA API**, so a release build that forgot its environment
flag would have shipped a store app talking to test data — riders booking rides that
do not exist, drivers accepting offers that count for nothing, with nothing on screen
to say so. Both now refuse to start in that state, and every non-production build
wears a visible badge.

Adding CI to the rider app — which had none — immediately found a test file that did
not compile and 19 tests that had never passed, including the entire ride-state
recovery suite. Those tests turned out to be untestable by construction:
`RideNotifier` built its network client inline, so the fake they carried was
unreachable. That is now fixed and nine of them run for the first time.

What did **not** get done matters as much: the rider app's three screen suites remain
unverified, the two operations consoles remain unreconciled, and the Cashfree payout
contract remains unproven. Those are the pilot's real risks, and none of them is a
code-volume problem.

## Current repositories / HEADs

| Repo | Stack | Branch | HEAD | CI | Tests |
|---|---|---|---|---|---|
| `SaaradhiGo-backend` | Django 5.2.17, Channels, Daphne 4.2.3 | `dev` | **`dc3546c`** | lint + SQLite suite + postgres-per-file + EC2 deploy | **732 collected**: 626 pass, 97 postgres-marked, 9 skipped |
| `SaaradhiGo-driver` | Flutter, Riverpod | `feat/driver-app-rebuild` | **`aad2ae6`** | analyze + test | 84 pass |
| `SaaradhiGo-mobile` (rider) | Flutter, Provider | `dev` | **`a505fd1`** | **added this run** | 28 pass, 19 named skips |
| `SaaradhiGo-web` (ops console) | Next.js 15.5.26 / React 19 | `develop` | `a4ec60b` | **none** | **none** |
| `SaaradhiGo-docs` | Markdown | `fix/promo-fare-integrity` | this document | n/a | n/a |

All working trees clean.

## Changes merged

**Backend** (`dev`)

| Merge | What |
|---|---|
| `1627827` | immutable fare snapshot schema + telemetry provenance |
| `56b9b6d` | `TripSettlement` immutable economic evidence |
| `7c55e6b` | `confirm_cash` acked a status the trip was not in — my own defect, found in QA |
| `dc3546c` | historical settlement classification (counts only, writes nothing) |

**Rider app** (`dev` @ `a505fd1`) — CI added; release builds can no longer target QA;
`assets/.env` no longer bundled; `rideServiceProvider` seam; four unused imports and
eight redundant null-assertions removed; 19 never-passing tests quarantined with
reasons.

**Driver app** (`feat/driver-app-rebuild` @ `aad2ae6`) — release builds refuse to
start when unconfigured or pointed at a non-production host; environment badge.

## QA deployments

Every backend merge deployed to Railway QA and was verified. Migrations applied
cleanly in order: `ride.0013`, `ride.0014_fare_snapshot_columns`,
`ride.0015_farepricing_trail_...`, `payments.0011_tripsettlement`. `/healthz` 200
throughout.

Two QA verification passes:

- **Command/idempotency pass** — 16/16 checks, repeated twice on different builds.
- **Settlement pass (new)** — 9/9 checks. First ride ever driven through cash
  confirmation. Earnings moved from 20 to 21 entries; the new entry reads amount
  124.40, commission 24.88, net 99.52.

## Rider journey

| Stage | | Evidence |
|---|---|---|
| Registration / login / OTP | GREEN | exercised in every QA ride |
| Token refresh | GREEN | 401-refresh-and-replay present; a 20-min ride outlives the 15-min token and the app handles it |
| Location permission / GPS | YELLOW | implemented; never executed on a device in this environment |
| Pickup selection | GREEN | precise-pickup screen exists |
| Destination search | GREEN | server-side maps proxy (geocode, place details) |
| Map / route preview | YELLOW | implemented; not visually verified |
| Vehicle selection | YELLOW | implemented; selected-state and rapid-change safety unverified on device |
| Fare estimate | GREEN | server-authoritative, `quote_fare` |
| Ride request | GREEN | idempotent on `client_request_id`; concurrent duplicates produce one trip |
| Searching UI / cancel | YELLOW | implemented; no-driver-found UX not verified on device |
| Driver assignment + details | GREEN | verified in QA (OTP reaches the rider, not the driver — correct) |
| Driver live location | GREEN | rider received all 120 positions on a 20-minute ride |
| Arrived / OTP display | GREEN | verified |
| Trip start | GREEN | OTP-gated, wrong OTP rejected with `invalid_otp` |
| In-trip map | YELLOW | implemented; not visually verified |
| SOS | GREEN | durable-first, deduped, 0.05s under 500 queued GPS frames |
| Support | GREEN | ticket create/list/message/close implemented |
| Completion | GREEN | committed exactly once, acked, survives retries |
| Fare display | YELLOW | total shown; the itemised breakdown widget exists and is **not wired** |
| Cash payment | GREEN | confirmed end to end in QA including settlement |
| Online payment | YELLOW | implemented; provider contract unverified |
| Receipt | YELLOW | implemented and versioned; retry/S3-failure behaviour not audited |
| Rating | YELLOW | endpoint exists; rider-visible surfacing unverified |
| Ride history | GREEN | endpoint + screen exist |
| Refund / dispute state | YELLOW | refund endpoint exists; no rider-facing dispute state |

## Driver journey

| Stage | | Evidence |
|---|---|---|
| Registration / OTP / refresh | GREEN | verified, including refresh-and-replay |
| Profile / KYC / documents / vehicle | GREEN | real audited approval exercised in QA |
| Approval state | GREEN | re-validated inside the accept lock, so a revoked approval cannot accept |
| Online / offline | GREEN | duty controller; server truth on reconnect |
| Location publishing | GREEN | 100% ingestion to 4 simulated hours |
| Offer UI / timeout / accept / reject | GREEN | 20s card matched to the dispatch wave; `trip_taken` handled |
| Double acceptance | GREEN | resolved under `SELECT FOR UPDATE` |
| Pickup navigation | YELLOW | maps present; navigation handoff unverified on device |
| Arrived / OTP verify / start | GREEN | verified in QA |
| Live trip + GPS | GREEN | 10-minute undrained-socket ride completed |
| SOS | GREEN | driver SOS never collapsed into the rider's — asserted |
| Complete | GREEN | acked `committed`; retry acked `already_done` |
| Cash confirmation | GREEN | **new**: runs settlement, verified in QA |
| Earnings / wallet / withdrawal | YELLOW | correct and read-only; still derived rather than reading `TripSettlement` |
| Ride history / ratings | YELLOW | present, unverified on device |
| Command ACK UX | GREEN | `CommandDispatcher` waits, retries the same id, never claims success without an ack, degrades to legacy frames |

## Operations journey

| Capability | | Evidence |
|---|---|---|
| Admin auth | GREEN | both consoles have login |
| RBAC | YELLOW | `IsAdmin` enforced on sensitive endpoints; no finer roles |
| Driver approval / KYC / vehicle review | GREEN | Django console, audited |
| Live trips / trip search / rider + driver search | GREEN | Django console (`global_search`, `fleet_monitor`) |
| SOS queue | GREEN | both consoles; acknowledge/resolve/false-alarm |
| Payment + payout lookup | GREEN | Django console; withdrawals screen in Next console |
| Refund / dispute workflow | YELLOW | `dispute_support` screen exists; workflow unverified |
| Pricing / zones | GREEN | rate cards immutable once created; zones screen |
| Promos | YELLOW | admin screen exists; promo **inactive** by design |
| Reports | YELLOW | `executive_revenue` recomputes commission at render time — drifts until it reads `TripSettlement` |
| Audit trail | GREEN | `admin_audit` app |
| System health | YELLOW | `/healthz` covers app + DB; no deeper operational check |
| **Console duplication** | **RED** | two consoles, both developed, no decision |

## UI/UX

Fixed this run: both apps refuse to run a misconfigured release build and show an
environment badge; the rider app's redundant null-defensiveness and dead imports
removed.

Not fixed, and deliberately not: I could not run either app (no Flutter SDK), so I
declined to blind-wire UI into a 2,000-line screen I cannot see. The highest-value
item found is `_FareBreakdown` — a complete fare-transparency panel (base, distance,
time, waiting, taxes, promo discount, total) that is never rendered. QA confirms the
API already returns matching fields. It is one wire-up from giving riders an itemised
fare instead of a bare total.

Nine other unused widgets sit in the same file from an in-progress refactor. They are
retained and documented rather than deleted, because deleting a finished feature as
part of a lint sweep is how work gets lost.

## Mobile

**Rider.** Went from no CI and unknown state to: analyze clean, 28 tests passing, 19
named skips. The skips are real debt, not cosmetic — three whole screens
(`personal_info`, `privacy_security`, `profile_tab`) have no working tests, and seven
ride-state-recovery tests are skipped with a diagnosis: two of them contradict each
other over the same SharedPreferences key, which points at mock state leaking between
tests rather than at the notifier.

**Driver.** 84 tests passing, analyze clean. Command ACK/retry/reconnect verified
including graceful degradation against a server with no ack support.

Neither app was built or run. Bundle IDs are inconsistent (`com.deeptrics.saaradhigo`
vs `com.saaradhigo.driver`). The rider app has a hardcoded Google Maps key committed
in its manifest; the driver app already does this correctly with `${MAPS_API_KEY}`,
so the right pattern exists next door and was not applied.

## Web/Admin

Next.js ops console: 14 screens, actively developed, **no tests, no typecheck script,
no CI, no deployment configuration**. Not touched this run beyond inspection. The
Django console is broader (19 templates, 6,619-line views) and includes aspirational
screens (`predictive_heatmaps`, `driver_loyalty`) that should not be read as
commitments.

## Backend

626 default tests passing, 97 postgres-marked, 732 collected. Ruff clean, bandit
high-severity zero, `manage.py check` clean, no pending migrations. The API surface is
120 routes across nine apps and is the most finished part of the platform.

## Ride lifecycle

GREEN. Strict transition table under `SELECT FOR UPDATE`; correlated
`command_ack` with committed / already_done / rejected; reconnect carries durable
status from PostgreSQL and is proven against a deliberately corrupted Redis cache;
booking idempotent on a per-rider key with a PostgreSQL partial unique index.

## GPS / Maps

GREEN for durability and boundedness: 100% ingestion at 30/60/120/240 simulated
minutes, exactly 12 durable points per minute regardless of transmit rate, per-trip
cap of 5,000 now actually enforced (it never triggered before — the check compared
against the current batch of 500). Raw trails are reachable by no API, pinned by
tests. Camera/marker behaviour on a device is unverified.

## Safety / SOS

GREEN. Durable row first, fan-out on commit to Celery, trip ownership verified,
nothing gated on Redis. Repeat presses collapse into one event with every rule
failing **open** — a different event type, an acknowledged original, a later repeat
and the other party on the trip all produce new events. Raw coordinates removed from
the SOS log. 0.05s response behind 500 queued GPS frames.

The escalation policy itself is a business decision that does not exist yet.

## Fare / Pricing

GREEN for provenance, by design not yet for billing. `FarePricing` now records
quote/final distinction, version, fare basis, quoted and actual quantities, min-fare
and surge-cap application, night surge separated from demand surge, rate-card version,
zone and vehicle **codes** rather than FKs, pricing source, gross/discount/rider
payable, promo identity, finalisation timestamp, and the telemetry that justifies the
basis (points, coverage ratio, max gap — added because the QA evidence showed coverage
ratio alone cannot carry that judgement).

`final_fare` is NULL everywhere and nothing writes it. Metered billing is off.

## Payments

YELLOW. Cash is proven end to end. Online payment is implemented but the provider
contract is unverified. Payment state is server-derived, never client-assumed.

## Settlement

GREEN as evidence, YELLOW as a source of truth, because nothing reads it yet.
`TripSettlement` is written inside `credit_driver_wallet`'s existing transaction and
shares the ledger's `TRIP_<id>_EARNING` idempotency boundary — five retries produce
one settlement, one ledger row, an unchanged balance.

One design deviation, made deliberately: the approved design specified `trip` as a
OneToOne *and* `version` for corrections as new rows, which cannot both hold.
Implemented as a FK with unique `(trip, version)` plus a partial unique index
guaranteeing exactly one row at `version=1`.

## Earnings

YELLOW. Correct and read-only, but still infers commission from the ledger's
direction. It should read `TripSettlement`; that is the next step and is what stops
admin revenue drifting.

## Payouts

RED, unchanged. The refund-on-ambiguous-failure path can plausibly pay a driver and
refund the platform. Attempts are capped at one as containment, not as an idempotency
fix. `cashfree-pg` is three major versions behind and its pins block 15 security
advisories — not upgraded, because that changes provider semantics and needs sandbox
proof plus authorisation.

## Receipts

YELLOW. Versioned, PDF, S3, resend endpoint, generated on commit so a failure cannot
roll back completion. The failure paths (S3 down, email down, repeated retries) were
not audited this run.

## Notifications

YELLOW. FCM push, in-app list and per-user preferences all exist. Which transitions
are guaranteed has not been established, so no minimum matrix is claimed.

## Support

GREEN for a pilot. Rider-facing ticket create/list/message/close plus admin
assign/reply, in both the API and the Next console.

## PostgreSQL

GREEN. All migrations this run additive: nullable columns, partial unique indexes,
check constraints. Constraint behaviour proven against real PostgreSQL rather than
SQLite, including under real-thread concurrency. No N+1 introduced — the per-trip
settlement count uses one `GROUP BY` per drain.

## Redis

YELLOW. Correctly non-authoritative and proven so: a deliberately corrupted
active-trip key does not change what the reconnect greeting reports. Degradation
behaviour under Redis loss was **not** tested this run. One anomaly remains
unexplained (below).

## Celery

YELLOW. Working; the per-task durability audit is still not done and Beat is still
embedded in the worker via `-B`.

## S3

GREEN for QA (scoped bucket, presigned access, no public KYC). Production storage
does not exist.

## Security

Dependencies: **56 → 16 vulnerabilities**, 9 vulnerable packages → 3, verified
against the full suite and 60 WebSocket/lifecycle tests on real infrastructure
(cryptography, daphne, Twisted, pyOpenSSL, PyJWT, setuptools). The remaining 15 share
one root cause: the Cashfree SDK's pins.

Fixed this run: release builds cannot target QA; SOS no longer logs coordinates; GPS
trails pinned unreachable. Outstanding: the rider app's committed Maps key, and a
full consumer-by-consumer IDOR and rate-limit audit.

## Privacy

GREEN for logging. SOS coordinates removed; GPS session summaries carry counts and
reason codes only, asserted by tests; classification output prints trip ids and is
asserted to leak no phone number or amount. `servers/redis_client.py` still logs raw
coordinates at DEBUG — inert at production log level, one environment variable from
not being.

## Performance

Lifecycle command latency flat 0.06–0.23s from 0 to 500 queued GPS frames. Per GPS
frame after earlier fixes: ~29 Redis commands end to end, 0.16 PostgreSQL
transactions, zero Celery enqueues, ~24ms. No load test beyond single-ride soaks —
concurrency numbers for 10/100/1,000 rides are **not** established and should not be
claimed.

## CI/CD

Backend: lint, SQLite suite, postgres-per-file, EC2 deploy. Driver: analyze + test.
Rider: **added this run**. Ops console: still none, and it is launch-critical.

## Observability

GREEN. Structured, PII-free events now cover: lost command sockets mid-ride, GPS
ingest failure (one warning plus a per-connection summary), location-frame coalescing,
ledger duplicate suppression, SOS raised and SOS repeat collapsed, trip request reused
and raced.

## Production infrastructure

Nothing exists. No production database, Redis, S3, secrets or deployment. Migration
policy is proven in QA (a failing migration blocks rollout; the previous healthy
revision keeps serving). Production deployment ownership — Railway or the EC2
workflow — is still undecided, so the EC2 workflow was deliberately left untouched.

## App-store readiness

Not submitted, as instructed. Known state: Android package IDs exist but are
inconsistent; permissions are declared and reasonable (the driver app requests
background location, which needs a Play Store justification); a hardcoded Maps key is
committed in the rider app; icons, splash, screenshots, privacy policy, terms,
support URL and account-deletion flow were not audited. DPDP data export and deletion
endpoints exist, which is the technical half of the account-deletion requirement.

## India compliance questions

Technical capability only. **None of these is legal advice and every one needs
professional confirmation.**

1. Aggregator / transport licensing for the pilot city and state.
2. Which driver documents are legally required versus merely collected, and their
   validity rules.
3. Commercial vehicle insurance requirements and who verifies them.
4. Privacy policy and terms of service — must exist before the apps are published.
5. GST treatment of the fare and of platform commission, and what a compliant invoice
   must contain. **Tax was deliberately not changed this run.**
6. Cancellation policy, including whether a fee is permitted. No fee is implemented
   and none is displayed.
7. Refund policy and timelines.
8. Grievance officer and published support channel.
9. Location-data retention period. The measured growth is 12 points per minute per
   ride; the policy is a business decision and no deletion mechanism should be built
   before it is made.
10. Emergency/SOS obligations — who must be contacted, how fast, and what must be
    recorded.
11. Driver classification and payout/settlement obligations.
12. Whether fare quoting without metering is permissible for the pilot.

## Pilot operations

Two runbooks written: `india-pilot-operations.md` (morning checks, incident playbooks,
deployment, rollback, evidence collection) and `india-pilot-scope.md`
(MUST/SHOULD/POST-PILOT against the actual product state). The capability map is in
`india-launch-capability-map.md`.

Both operations documents state the same rule first: **never edit a financial table by
hand.**

## External blockers

| Blocker | Consequence |
|---|---|
| No Cashfree sandbox credentials | Payout duplicate behaviour unverifiable; 15 security advisories unreachable |
| No Flutter/Dart SDK in this environment | Neither app could be run; all mobile UX findings are source-level or CI-level |
| No QA shell or admin credential | The historical classification command could not be run against real data |
| No production accounts | No production infrastructure could be prepared beyond documentation |
| Tax, insurance, licensing, retention policy | Business/legal decisions engineering cannot supply |

## Genuine pilot blockers

Five. Everything else in this document is ordinary work.

1. **Payout correctness depends on an unverified provider contract.** A driver could
   plausibly be paid and the platform refunded for the same withdrawal. Cash-first
   reduces the exposure; drivers still withdraw.
2. **Two operations consoles, no decision.** During an incident, an operator acting in
   one and looking in the other will conclude the platform is broken.
3. **No production infrastructure.** Database, Redis, S3, secrets and deployment do
   not exist.
4. **The ops console has no CI, no tests and no typecheck**, and it is the tool
   operations depend on during a live pilot.
5. **Three rider screens have no working tests.** Profile, privacy and personal
   information are unverified, and the rider app is the less-maintained half of a
   two-sided product.

## Technical debt

Separate from the above, and none of it blocks a pilot: Celery Beat still embedded;
per-task durability audit outstanding; earnings and admin revenue still derive rather
than read `TripSettlement`; historical settlements un-backfilled; receipt failure
paths unaudited; Redis degradation untested; consumer-by-consumer IDOR/rate-limit
audit outstanding; `redis_client` DEBUG coordinate logging; bundle-ID inconsistency;
rider app's committed Maps key; 8 of 97 postgres tests fail in one process (CI runs
per-file, 97/97); the rider `SharedPreferences` test-leak diagnosis.

## Human actions required

1. **Decide which operations console is production** and stop developing the other.
2. **Obtain Cashfree sandbox credentials** and run the existing payout checklist.
   This gates both payout safety and 15 security advisories.
3. **Name the production deployment owner** — Railway or EC2.
4. **Provision production infrastructure**: PostgreSQL with backups, Redis with a
   stated persistence and eviction policy, a private encrypted S3 bucket with
   least-privilege credentials, and the secret set.
5. **Supply the business/legal decisions** in the compliance list above — at minimum
   privacy policy, terms, GST treatment, cancellation policy, SOS escalation policy
   and GPS retention period.
6. **Run** `python manage.py classify_historical_settlements` in QA and production
   (read-only) before authorising any settlement backfill.
7. **Rotate the Google Maps key** committed in the rider app's manifest, and restrict
   it by package name and signing certificate.
8. **Verify the release-build guard** once per app: `flutter build apk --release`
   with no flags must refuse to start; with production flags it must run.

## India pilot readiness matrix

| Area | | Evidence |
|---|---|---|
| Rider Android | YELLOW | builds and has CI now, but three screens have no working tests and it was never run on a device here |
| Driver Android | GREEN | 84 tests green in CI, full lifecycle verified in QA including a 10-minute ride |
| Rider iOS | RED | not targeted; no iOS build or configuration verified |
| Driver iOS | RED | not targeted |
| Rider authentication | GREEN | OTP + refresh exercised in every QA ride |
| Driver authentication | GREEN | plus 401-refresh-and-replay in the client |
| KYC | GREEN | real audited approval path exercised in QA |
| Maps | YELLOW | server-side proxy holds the key; rendering unverified on device; rider key committed in git |
| Location | YELLOW | ingestion proven server-side; permission flows unverified on device |
| Ride request | GREEN | idempotent; concurrent duplicates produce one trip on real PostgreSQL |
| Dispatch | GREEN | Celery waves, per-driver fan-out that drops rather than blocks |
| Acceptance | GREEN | race resolved under row lock; approval re-validated inside it |
| Lifecycle | GREEN | acked post-commit, correlated, retry-safe, 0–500 frames flat |
| Reconnect | GREEN | durable status from PostgreSQL, proven against a corrupted cache |
| GPS | GREEN | 100% ingestion to 4 simulated hours; 12 points/min; per-trip cap now enforced |
| SOS | GREEN | durable-first, fails open, 0.05s under 500 queued frames |
| Fare quote | GREEN | single server-side engine; client never trusted |
| Fare persistence | GREEN | immutable snapshot with provenance; nothing reads it yet by design |
| Cash | GREEN | proven end to end including settlement — QA trip 40 |
| Online payment | YELLOW | implemented; provider contract unverified |
| Settlement | GREEN | immutable, idempotent, concurrency-proven; not yet read by reports |
| Earnings | YELLOW | correct and read-only, still derived rather than reading settlement |
| Payout | RED | unverified provider contract; duplicate-payment path plausible |
| Receipt | YELLOW | works and cannot roll back completion; failure paths unaudited |
| Notifications | YELLOW | channels exist; guaranteed transitions not established |
| Support | GREEN | rider tickets + admin reply, API and console |
| Admin | YELLOW | functionally broad; split across two consoles |
| Operations | RED | two consoles, no decision, no CI on the newer one |
| PostgreSQL | GREEN | additive migrations, constraints proven under real concurrency |
| Redis | YELLOW | correctly non-authoritative; degradation untested; one unexplained anomaly |
| Celery | YELLOW | works; durability audit outstanding, Beat embedded |
| S3 | YELLOW | QA proven; production absent |
| Security | YELLOW | 40 advisories closed; 15 blocked by one SDK pin; WS IDOR audit outstanding |
| Privacy | GREEN | no PII in logs, asserted by tests; one latent DEBUG coordinate log |
| CI | YELLOW | backend, driver and rider covered; ops console has none |
| Deployment | RED | no production deployment and no named owner |
| Monitoring | YELLOW | rich structured events; no alerting or dashboards |
| App Store | RED | nothing submitted; store assets and policies unaudited |
| Legal/business | BLOCKED | twelve open questions requiring professional confirmation |

## Top 10 remaining launch actions

Ranked by launch risk, not by engineering appeal.

1. Verify the Cashfree payout contract in sandbox — it is the only path to money
   being wrong in a way nobody can undo.
2. Choose one operations console and make it the only one.
3. Provision production infrastructure and name its owner.
4. Get the business/legal decisions: privacy policy, terms, GST, cancellation, SOS
   escalation, GPS retention.
5. Add CI and tests to the ops console.
6. Fix the rider app's three untested screens on a device.
7. Point earnings and admin revenue at `TripSettlement`, then backfill A+B categories
   marked `reconstructed`.
8. Rotate and restrict the rider Maps key.
9. Audit Celery task durability and separate Beat.
10. Wire the fare breakdown into the rider UI — cheapest trust win available.

## Next engineering run

1. Point driver earnings and admin revenue at `TripSettlement` and delete the
   derivation code; run the classification command and backfill A+B as
   `reconstructed`.
2. Add CI, typecheck and a first test suite to the Next.js ops console, then decide
   the console question with evidence.
3. Fix the rider app's three quarantined screen suites and the SharedPreferences
   test-leak, on a machine with a Flutter SDK and an emulator.
4. Audit Celery task-by-task (idempotent? financial? safe to redeliver?), separate
   Beat, and test worker loss for dispatch, GPS persistence and receipts.
5. Test Redis degradation against an active ride, and root-cause the single-process
   PostgreSQL anomaly so the CI suite can run in one process again.

## If SaaradhiGo launched tomorrow in one controlled Indian pilot city

Answered from evidence, not from confidence.

**A driver could lose money and nobody could prove what happened.** This is the worst
one. A withdrawal that fails ambiguously can, on the current code path, refund the
platform *and* leave a transfer with the provider. Nobody has verified Cashfree's
actual behaviour, so the size of this is unknown rather than small. Attempts are
capped at one as containment. A driver in that situation gets "we are checking with
the bank" and no timeline.

**An operator could act in the wrong console during an incident.** Two consoles
exist, both work, and they do not cover the same things. Approving a payout in one
and looking for it in the other is a realistic five-minute confusion at exactly the
wrong moment.

**A rider could hit a broken screen.** Profile, privacy and personal-information
screens have no working tests — their test suites have never passed. I could not run
the app, so I do not know whether they are fine or broken; I know only that nothing
verifies them.

**A rider sees a fare with no explanation.** They get a total. The itemised breakdown
exists in the code and is not connected. In a cash market where the driver is handed
notes, "why is it this much" is a daily conversation, and the app cannot answer it.

**Operations cannot see the platform.** There is no alerting and no dashboard. The
structured events needed to diagnose a problem now exist, but somebody has to go
looking; nothing tells them to look.

**Nothing is deployed to production.** No database, Redis, S3, secrets or deployment.
"Launching tomorrow" is not currently possible in the literal sense.

**A driver's phone could be on QA.** Fixed this run — a misconfigured release build
now refuses to start and non-production builds are badged — but it must be verified
once per app with a real release build before anyone ships.

**What would probably go right:** the ride itself. The lifecycle is the most tested
part of this platform. Commands are acknowledged after commit, retries are safe,
reconnects restore durable truth, GPS survives four hours of simulated driving, cash
settles exactly once with arithmetic that holds, SOS answers in 0.05s under heavy
telemetry, and `final_fare` stays NULL so nobody is billed by a half-built metering
path. A rider getting a car and a driver getting paid the right amount is the part I
would bet on.

The parts I would not bet on are the parts around it: the provider we do not
understand, the console we have two of, the screens nothing tests, and the production
environment that does not exist.

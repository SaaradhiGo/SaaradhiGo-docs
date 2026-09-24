# India production launch closure report

2026-09-24. Parallel-workstream run: every workstream touched, then the highest-risk
findings deepened.

No secret value appears anywhere in this document. Variable, secret and key **names**
appear because the gap is which names exist, not what they hold.

---

## Executive summary

Eight real defects were found and fixed. Five of them share one shape: **a broad
exception handler converting an infrastructure failure into a confident business
answer.** That pattern is now the single most important thing known about this
codebase.

| # | Defect | Why it mattered |
|---|---|---|
| 1 | **Both mobile apps were unbuildable**, for two unrelated reasons. The driver app's `AndroidManifest.xml` contained `--` inside an XML comment, so it did not parse and every APK build failed. The rider app's `signingConfigs` block cast a null to `String` whenever `key.properties` was absent, killing even `--debug`. | No driver or rider could ever have installed either app. 83 and 68 tests passed throughout, because neither `flutter test` nor `flutter analyze` reads Gradle or the manifest. Both were found within minutes of adding a build gate. |
| 2 | **An unknown payout outcome refunded the driver's wallet.** `create_upi_payout` returned `None` for a validation failure, a missing token, and a network failure *after* the request was issued. The caller marked the withdrawal `failed` and credited the balance back. | A 15-second timeout could pay a driver twice: once by Cashfree, once by the refund. |
| 3 | **The OTP task reported success without sending.** A bare `@shared_task` caught every failure and returned a dict, so Celery recorded SUCCESS and there was no retry at all. | The first step of every rider and driver session. One SNS throttle meant the OTP silently never arrived. Also used by SOS. |
| 4 | **Five Celery tasks were acknowledged before running.** Neither `acks_late` nor `reject_on_worker_lost` was set. | A killed worker silently dropped `auto_cancel_trip` (rider waits forever) and `reconcile_stuck_withdrawals` (stuck payouts stop being surfaced). |
| 5 | **A committed trip was reported as a failed booking.** `auto_cancel_trip.apply_async` sits outside the atomic block, inside a broad handler; a Redis outage committed the trip and then returned "creation failed". | An orphan `requested` trip the rider was told did not exist, with no timeout scheduled to clean it up. |
| 6 | **A privacy toggle persisted fire-and-forget.** `unawaited(setTwoFactorEnabled(...))`. | A rider could turn two-factor off and lose the change if the app died before the write landed. |
| 7 | **Production would boot with QA's auth bypass.** Nothing stopped `TEST_PHONE_NUMBERS` or `QA_ADMIN_BOOTSTRAP_*` reaching production — and the remediation plan is literally "copy QA's variables". | Fixed-OTP login and an admin back door into payout approval and KYC. |
| 8 | **Revision visibility was cosmetic.** `/healthz` carried a `version` read from a hand-set static variable, identical across every deploy. | This is what produced a false verification failure last run: a 200 from the old container read as proof of the new one. |

**The verdict has not changed, and one part of it got worse.** Production still
cannot boot and still tracks `dev`. The leaked database password is still not
rotated. And the previous report's "no APK has been produced" was understating it:
no APK *could* have been produced.

Backend gates went 644 → **715 passing**. That number is evidence about specific
behaviours, not a readiness score.

---

## Exact repo HEADs

| Repository | Branch | HEAD |
|---|---|---|
| SaaradhiGo-backend | `dev` | `7d609b1` |
| SaaradhiGo-mobile (rider) | `dev` | `34b3069` |
| SaaradhiGo-driver | `feat/driver-app-rebuild` | `bae0a3e` |
| SaaradhiGo-web (ops) | `develop` | `a798982` |
| SaaradhiGo-docs | `docs/india-launch-readiness` | this commit |

The docs branch was renamed from `docs/rider-test-debt`, which had become
misleading. No commits were lost; the old ref is still on the remote and can be
deleted once the rename is noticed.

## Deployment revisions

| Environment | Service | Tracks | State |
|---|---|---|---|
| Railway QA | `backend` | `dev` | Deploys succeed. Last verified live revision `2499720`. |
| Railway production | `SaaradhiGo-backend` | **`dev`** | `crashed: 1, running: 0`. Recent deploys FAILED. Last successful build crash-looped on `ALLOWED_HOSTS`. |
| GitHub Actions `deploy` (EC2) | — | `dev` | Has never succeeded. `EC2_HOST_KEY` does not exist and the host does not answer on port 22 from the runner. |

---

## Workstream A — Production / Release

**Reviewed.** Railway project (2 environments, 7 services), both environments'
variable name sets, GitHub workflows, branch deployment rules, repository secret
names, production deploy logs and diagnosis.

**Changed.** Added `GET /version` and extended `/healthz` with resolved revision
metadata. Added production configuration guards that refuse to boot on DEBUG=True,
a missing `DJANGO_SECRET_KEY`, `TEST_PHONE_NUMBERS`, any `QA_ADMIN_BOOTSTRAP_*`, or
QA/staging/localhost URLs — all faults reported together.

**Tests.** 26. The boot guards are exercised as **real boots in a subprocess**,
because the property is "the process refuses to start". Every guard has a negative
control, including one asserting a correct production configuration *does* boot —
which is simultaneously the **A5 production boot rehearsal**, proving the required
variable set is sufficient with dummy values and touching nothing real.

**QA evidence.** Deployment truth established from the Railway API, not inferred.
Production holds **8 variables, all `RAILWAY_*` injected**, against QA's 30.

**Remaining risk.** Production is unconfigured, tracks `dev`, and has a staged
`EnvironmentPatch` pending since 2026-09-22. Repointing the service and populating
secrets are infrastructure mutations requiring the account owner — **not performed**.
Why production's recent *builds* fail (as distinct from the startup crash) is
unresolved.

### A2 — release model

Simplest safe model, deliberately not GitFlow:

```
feature/*  ──▶  dev  ──▶  (QA auto-deploy, acceptance)  ──▶  tag v*  ──▶  production
```

- `dev` stays the integration branch and keeps auto-deploying to **QA only**.
- Production deploys from an **immutable tag** (`v2026.09.25-1`), never a branch.
  A tag cannot acquire new commits, which is the whole property `dev` lacks.
- Promotion is a human action: create the tag, then trigger the production deploy.

This needs no new branch and no new tooling. It needs production repointed off
`dev`.

---

## Workstream B — Security / Secrets

**Reviewed.** Git history for the committed database password; all five repositories'
working trees scanned for nine credential patterns; `cashfree-pg` constraint chain;
OTP throttling; the admin login and `admin_required`.

**Changed.** Nothing rotated — rotation requires the account owner. Production guards
(above) now block the QA auth bypass from reaching production.

**Tests.** Covered by the boot-guard suite.

**QA evidence.** Scan produced 9 candidates, classified without printing values:

| Finding | Classification | Action |
|---|---|---|
| `POSTGRES_PASSWORD`, commit `9af854c`, 2026-05-07, reachable from `dev` + 5 branches | **ACTIVE — compromised** | Rotate. ~4.5 months exposed, in every clone. |
| Google **Maps** API key, `AndroidManifest.xml` + `AppDelegate.swift` — **one key shared by Android and iOS** | **ACTIVE — compromised** | Rotate and split per platform. A single key cannot be platform-restricted on both, so it is almost certainly unrestricted and billable by anyone who extracts it from the APK. Already flagged in an older runbook and still not done. |
| Firebase client config (`google-services.json`, `firebase-config.js`) | Public by design | Ships in every binary. Not a secret; Firebase security rules are what matter. |
| `tests/test_s3_endpoint_support.py` AWS key | Test/fake | Its own comment says "credential-shaped values… without real ones". None. |
| JWT and password literals in 3 test files | Test/fake | None. |
| `redis://:<password>@host` in a runbook | False positive | A template placeholder, not a credential. |

**B4 — git history.** Do **not** rewrite yet. Rotation ends the exposure; rewriting
does not, because every existing clone keeps the value. Rewriting first would create
the impression the problem was handled while a live credential stayed in circulation.
After rotation, history cleanup is optional tidying.

**Remaining risk.** Two active credentials unrotated. Secrets are **not sealed** in
either Railway environment (`sealedVariableNames: []`), so every QA secret is an
ordinary readable variable. The admin login still has **no rate limiting and no
MFA**.

### B6 — ops MFA

Do not build custom MFA. The practical launch option is to put the Django console
behind an identity proxy (Cloudflare Access or equivalent) so MFA is enforced before
a request reaches Django, requiring no application change. Failing that,
`django-otp` + `django-two-factor-auth` is the conventional in-framework choice.
**Exact launch requirement:** payout approval and KYC approval must require a second
factor. Neither is implemented.

---

## Workstream C — Rider Mobile

**Reviewed.** Both unresolved account-surface questions; the quarantine; build
feasibility.

**Changed.** Privacy toggles now `await` their own persistence. Rider CI now builds a
debug APK. Gradle signing config made conditional so a build is possible at all.

**Tests.** 68 passing, 7 skipped (was 67/8). Every skip has an explicit reason.

**QA evidence.** Both open questions resolved by reproduction:

- **Privacy toggle** — primarily a **test defect**: the tap targeted
  `_PrivacyToggleRow`, whose centre is in its title text, so it missed the Switch
  entirely. `ensureVisible` alone was not enough. Found alongside it: a **real**
  unawaited-persistence defect, fixed.
- **Profile save** — a **test defect**: it asserted a phone number it never
  arranged, with empty preferences and a fake API returning only `full_name`. Now
  seeded, so the assertion means what it was meant to.

**Remaining risk.** One test still skipped, for a narrower and honestly-stated
reason: a "deactivated widget's ancestor" error during the post-save route
transition. The screen guards every post-await `context` use with `mounted`, so this
is probably the stubbed router meeting `usePhoneSurface`'s teardown — **but I did not
confirm it, and the skip reason says so.** C5 (full journey widget tests) and C6
(error/offline states) were **not done**.

---

## Workstream D — Driver Mobile

*Not reviewed at all in the previous run. The most valuable single finding of this
one came from here.*

**Reviewed.** Test and analyze baseline; Android manifest; Gradle config; Firebase
push wiring; command-ACK implementation.

**Changed.** Fixed the manifest so the app can be built. Added an APK build job.
Added `test/android_manifest_test.dart`.

**Tests.** **83 passing**, `flutter analyze`: no issues. Plus 5 new manifest tests.

**QA evidence.** The build job failed on its first run with
`ManifestMerger2$MergeFailureException: Error parsing AndroidManifest.xml`. Root
cause: a comment spelling out `flutter build apk --dart-define=...`, and `--` is
illegal inside an XML comment. **The driver app had never been buildable.**

Command ACK (D3) is genuinely well built — `CommandDispatcher` with correlated
`command_id`, `committed`/`already_done`/`rejected`, and legacy-frame fallback, all
covered by existing tests including a full reconnect state machine.

**Remaining risk — launch-blocking.** The app depends on `firebase_messaging`, but
`com.google.gms.google-services` is **commented out** because
`google-services.json` is absent. `Firebase.initializeApp()` throws,
`PushService` catches it and sets `_ready = false`, and the app registers **no FCM
token**. A driver whose app is backgrounded **receives no ride offer**. D4 (token
expiry during a long ride) and D5 (background GPS on a real device) were **not
tested**.

---

## Workstream E — Operations

**Reviewed.** The canonical-console decision; the repaired KYC driver-list route;
the SOS operator endpoint.

**Changed.** Nothing new this run. The Django console remains canonical (decision
and evidence in `operations-console-decision.md`); the Next.js console's 404 and its
CI were fixed in the previous run.

**Tests.** Ops console: lint, type-check, build — all green in CI.

**QA evidence.** `/api/v1/driver/admin/` → 401 (exists, protected);
`/api/v1/driver/admin/list/` → 404. All 19 console endpoints probed; 18 were already
correct. SOS operator listing verified to read PostgreSQL directly, which is what
makes it the fallback when paging fails.

**Remaining risk.** The alternate console is **not yet marked DEPRECATED in the UI**.
E4 (time-in-state for active rides), E6 (trip finance investigation view), E7
(support ticket routing) and E8 (confirmation/authorisation/audit on dangerous
actions) were **not done**. No operator has been told in writing which console to
use.

---

## Workstream F — Ride / Redis / WebSocket

**Reviewed.** 57 broad-handler fail-open returns across the backend, classified by
domain. WebSocket auth, ownership and trip-group membership. Trip creation's
transaction boundary.

**Changed.** Trip creation no longer reports a committed trip as a failed booking.
(Last run: `get_driver_active_trip` no longer reports a Redis failure as "driver is
free".)

**Tests.** 4 new coherence tests, 9 existing for the Redis fail-open.

**QA evidence — the F3 audit, by file:**

```
25  servers/redis_client.py          1 fixed (driver availability)
 6  cashfree_gateway.py              1 fixed (payout outcome) — the highest-risk file
 5  servers/consumers.py             1 fixed (trip creation)
 3  servers/ride/dispatch.py         classified, not changed
 3  servers/driver/services.py       1 fixed (inner handler swallowing the payout signal)
 3  servers/ride/receipts.py         classified, not changed
 3  auth_user/dpdp_views.py          classified, not changed
```

**F4 is genuinely solid and needed no work**: `test_losing_driver_is_denied_the_trip_socket`,
`test_unassigned_driver_is_a_candidate_not_a_subscriber`,
`test_rider_is_always_a_participant` and `test_trip_driver_details_rejects_unrelated_user`
already cover cross-trip IDOR, and the consumer restricts the trip group to the rider
and the **assigned** driver.

**Remaining risk.** F1's full outage matrix (Redis down before booking, during
dispatch, after assignment, mid-trip, before completion) was **not** exhaustively
exercised — only the trip-creation and SOS paths. **F2 was not run**: driver
exclusivity under concurrent acceptance *with Redis unavailable* remains unproven,
and the DB row lock is the last defence. F5 (GPS traffic not starving SOS/complete/
cancel) was not measured.

---

## Workstream G — Celery / Async

*Not reviewed in the previous run.*

**Reviewed.** Every registered task, the beat schedule, ack semantics, prefetch,
time limits, and the OTP delivery path.

**Changed.** `CELERY_TASK_ACKS_LATE`, `CELERY_TASK_REJECT_ON_WORKER_LOST`,
`worker_prefetch_multiplier = 1`, and soft/hard time limits. OTP delivery now
retries transient failures with backoff and keeps terminal failures terminal.

**Tests.** 24. Including an inventory guard that **immediately earned its place** by
finding a 13th task my own AST scan had missed — `base.utils.send_otp_via_sns`,
because it lives in `base/` rather than `servers/`.

**G1 — task inventory (13):**

| Task | Trigger | Financial | Durable now |
|---|---|---|---|
| `base.utils.send_otp_via_sns` | on demand (every login, SOS) | no | yes + retry (**added**) |
| `auth_user.send_push_notification_task` | on demand | no | yes |
| `servers.ride.tasks.auto_cancel_trip` | ETA, at trip creation | no | yes (**was not**) |
| `ride.compute_trip_actuals` | ETA after completion | indirectly | yes |
| `ride.dispatch_wave` | ETA, per wave | no | yes |
| `ride.issue_receipt_for_trip` | on commit | tax document | yes |
| `ride.persist_location_trail` | beat, 60s | no | yes |
| `sos.dispatch_sos` | on commit | no | yes |
| `pricing.fare_shadow_sweep` | beat, 15m | no (shadow only) | yes |
| `driver.block_expired_driver_licenses` | beat, daily 02:00 | no | yes (**was not**) |
| `driver.sweep_stale_driver_presence` | beat, 60s | no | yes (**was not**) |
| `payments.reconcile_stuck_payments` | beat, 5m | **yes** | yes (**was not**) |
| `payments.reconcile_stuck_withdrawals` | beat, 10m | **yes** | yes (**was not**) |

**Remaining risk.** **Beat is still embedded in the worker via `-B`** — safe only
while exactly one worker runs, and it must be separated before scaling. G2's
kill-a-real-worker simulation was **not** performed; durability is asserted from
configuration, which is honest but weaker than an interruption test. G3 (receipt
retry semantics) was not exercised end to end.

---

## Workstream H — Finance / Payments / Payouts

**Reviewed.** Payout create/retry/refund paths end to end; the Cashfree gateway's
failure classification; `WithdrawalRequest` states; existing settlement and
reconciliation tests.

**Changed.** The gateway now distinguishes a **definite rejection** from an
**unknown outcome**. An unknown outcome sets a new `unresolved` state, does **not**
refund, records no provider reference, and cannot be re-dispatched. An inner broad
handler that swallowed the new signal now re-raises it. Migration `0022` is additive.

**Tests.** 11, as reproduction → negative control → regression. The controls matter
as much as the fix: a **definite** rejection must still refund, or "stop refunding"
would strand every driver whose payout was legitimately declined.

**QA evidence.** H5 containment was already proven by
`test_payout_single_provider_call.py` — exactly one provider create call per
execution — and that file's own docstring had named this exposure as unfixable
without the provider contract. It was fixable: ambiguity is determinable client-side.

**Remaining risk.** H1 (rerun settlement concurrency and historical immutability),
H2 (money precision boundary tests at ₹0.01 and large fares) and H3 (another full QA
cash ride) were **not done this run**. Resolving an `unresolved` payout is a manual
console action that **does not yet exist** — the state can be entered and not left.

### H6 — Cashfree sandbox verification suite (required before any upgrade)

`cashfree-pg` is pinned at **3.2.12**; latest is **6.0.1** — three major versions.
Before upgrading, prove in sandbox:

1. `create_upi_payout` succeeds and returns a reference.
2. The **same `transferId` replayed** is rejected as a duplicate and does **not**
   create a second transfer. *(This is the assumption the whole retry story rests on
   and it is currently unverified.)*
3. A payout to an invalid VPA returns a definite 4xx rejection, not a 5xx.
4. Behaviour on a forced timeout — whether the transfer later appears.
5. A status/fetch call for a known `transferId`, if one exists in 6.x.
6. Webhook signature verification against 6.x payloads.
7. Order create, capture and refund for the rider payment path.
8. Full regression of the 715-test backend suite against the new SDK.

Until 2 is proven, **no automatic retry of an ambiguous payout is safe.**

---

## Workstream I — Safety / SOS / Support

*Not reviewed in the previous run.*

**Reviewed.** Rider and driver SOS entry path, dedup, durability, fan-out ordering,
coordinate privacy, operator visibility.

**Changed.** Nothing — the implementation was already sound. Added tests.

**Tests.** 6 integration tests under a simulated broker outage.

**QA evidence.** SOS is **HTTP + PostgreSQL**, not socket-dependent: the row commits
in a transaction, the fan-out is deferred with `transaction.on_commit` so a rollback
cannot page anyone about a phantom emergency, dedup is decided in PostgreSQL
specifically so a cache miss cannot double-record and a cache hit cannot drop one,
and trip ownership is enforced. Coordinates are deliberately absent from log lines —
asserted by scanning emitted records, not by reading the code.

**Remaining risk — and it is a real one.** `dispatch_sos.delay()` failing is caught
and logged. With Redis down an SOS is **durable but unannounced**: the rider is told
"Help is on the way" and nobody is paged. The only thing between a recorded SOS and
an unnoticed one is **an operator actively watching the console**, which is therefore
a launch requirement rather than a nicety. I4's other conditions (SOS under heavy GPS
load, during socket reconnect) were not exercised. I6 support verification not done.

---

## Workstream J — Data / GPS / Privacy

*Not reviewed in the previous run.*

**Reviewed.** `TripLocationPoint` schema and indexes, sampling policy, Redis stream
bounds, the consumer-group drain, access control.

**Changed.** Nothing. This subsystem is in better shape than the rest.

**Tests.** 10 existing access-control tests, including cross-rider and cross-driver
IDOR and an assertion that no serializer, admin registration or route exposes the
trail at all.

**J1 — storage model.** Row ≈150 bytes of heap; five indexes (PK, trip+time,
driver+time, received_at, unique trip+source_event_id) ≈150 bytes. **≈250–400 bytes
per point.** Sampling floors at 5s/25m with a 12 points/minute ceiling and a
5,000-point per-trip cap, so a 15–25 minute urban ride is **≈200–300 points**, or
**50–120 KB per ride**.

| Rides/day | Per day | Per month | Per year |
|---|---|---|---|
| 100 | 5–12 MB | 0.15–0.36 GB | 2–4 GB |
| 1,000 | 50–120 MB | 1.5–3.6 GB | 18–44 GB |
| 10,000 | 0.5–1.2 GB | 15–36 GB | 180–440 GB |
| 100,000 | 5–12 GB | 150–360 GB | 1.8–4.4 TB |

A controlled pilot is negligible. At 10,000 rides/day this becomes the largest table
in the database and needs partitioning by month.

**J4 — streams are bounded.** `driver_location_stream` `maxlen=100000`,
`ride_requests` `maxlen=50000`. The drain uses a consumer group, bounds each read and
each run, and **acks off-trip events** so the pending list cannot grow without limit.
No unbounded stream.

**J5 — client timestamps are already handled correctly.** `recorded_at` is the
device clock and documented as untrusted; `received_at` is ours and is what anything
legal or financial reasons about. Both are kept "precisely because they disagree".

**Remaining risk.** `GPS_TRAIL_ENABLED` defaults to **False** — so there is
currently **no durable route evidence** for disputes, SOS investigation or insurance.
That is a deliberate pilot decision to make, not a bug. J2 (retention policy) is
**not proposed or implemented**, correctly, because deletion policy needs business
and legal approval. J5's boundary tests (future, ancient, out-of-order, duplicate,
clock-skewed timestamps) were not written.

---

## Workstream K — CI / Dependencies / Quality

**Reviewed.** Backend suite isolation; dependency vulnerabilities and their
constraint chain; all four repositories' CI.

**Changed.** Rider and driver CI now **build an APK**. (Last run: postgres suite
collapsed from ten processes to two; ops console CI created.)

**Tests.** Backend **715 passing**, 9 skipped; all 97 postgres-marked tests pass in
two processes, verified green in CI.

**K1.** The one remaining single-process failure is unchanged and still narrowed to a
**2-file, 100-second reproduction** (`test_lifecycle_under_gps_load.py` after
`test_driver_active_trip_contention.py`). Timeboxed and not reopened this run.

**K2 — dependency audit.** 14 vulnerabilities in 2 packages:

```
urllib3     2.0.7   6 distinct CVEs   fixes need 2.2.2 / 2.5.0 / 2.6.0 / 2.6.3 / 2.7.0
sentry-sdk  1.32.0  1 CVE             fix needs 1.45.1
```

**All 14 are blocked by one pin.** `cashfree-pg==3.2.12` declares
`urllib3 <2.1.0` and `sentry-sdk <1.33.0`. `botocore` and `requests` both permit
`urllib3 <3`, so they are not the constraint. **Independently fixable: zero.** The
entire dependency posture is gated on the Cashfree upgrade, which is gated on the
sandbox suite in H6.

**Remaining risk.** K6 (docs link checking) not done. The GitHub Actions `deploy`
job still fails on every backend push, so **the backend pipeline is permanently red**
while `test`, `lint` and `postgres-concurrency` are all green — which teaches people
to ignore the run that matters.

---

## Workstream L — End-to-End Pilot

**Reviewed.** Existing acceptance coverage.

**Changed.** Nothing.

**Tests.** None added.

**QA evidence.** **None. L was not executed.** No canonical pilot acceptance ride was
run this session, no negative acceptance cases were exercised, and no trip ID,
settlement ID or financial values were captured.

**Remaining risk.** This is the largest untouched workstream. The prior run drove a
full QA cash ride including cash confirmation, so the path is known to work
end to end at that commit — but **not at `7d609b1`**, and this run changed Celery ack
semantics, trip creation error handling, payout classification and the OTP task.
Those are exactly the changes an acceptance ride exists to catch.

---

## Production architecture

```
                    ┌──────────────┐        ┌──────────────┐
   Rider APK ──┐    │  Rider app   │        │  Driver app  │    ┌── Driver APK
   (CI, debug) │    │   (Flutter)  │        │   (Flutter)  │    │   (CI, debug)
               │    └──────┬───────┘        └──────┬───────┘    │
               │           │ HTTPS + WSS           │ HTTPS + WSS
               │           ▼                       ▼
               │    ┌──────────────────────────────────────┐
               │    │        Daphne / Django ASGI          │
   Ops console ├───▶│  REST  /api/v1/**                    │
   (Next.js,   │    │  WS    /ws/driver, /ws/rider, /ws/trip│
    or Django  │    │  Ops   /  (Django console, canonical) │
    console at │    │  Health /healthz   Revision /version  │
    the root)  │    └───┬──────────┬──────────┬─────────────┘
               │        │          │          │
               │        ▼          ▼          ▼
               │  ┌──────────┐ ┌───────┐ ┌──────────┐
               │  │PostgreSQL│ │ Redis │ │   S3     │
               │  │  TRUTH   │ │ db0 broker      │  │ KYC docs │
               │  │          │ │ db4 channels    │  │ receipts │
               │  └──────────┘ │ geo / presence  │  └──────────┘
               │               │ streams (bounded)│
               │               └────────┬─────────┘
               │                        │
               │              ┌─────────▼──────────┐
               │              │ Celery worker      │
               │              │  + beat (EMBEDDED  │
               │              │    via -B — must   │
               │              │    be separated)   │
               │              └─────────┬──────────┘
               │                        │
               │              ┌─────────▼──────────┐
               └─────────────▶│ AWS SNS (OTP SMS)  │
                              │ FCM (push)         │
                              │ Cashfree (pay/out) │
                              └────────────────────┘

PostgreSQL is authoritative. Redis is live state and a broker; when it fails the
system must degrade, never invent a business answer. Three of this run's eight
defects were exactly that invention.
```

## Production environment gaps

Production holds **8 variables, all Railway-injected**. QA holds **30**. Missing, all
required: `ALLOWED_HOSTS`, `DJANGO_SECRET_KEY`, `DB_HOST/NAME/USER/PASSWORD/PORT/SSLMODE`,
`REDIS_URL`, `JWT_SIGNING_KEY`, `CASHFREE_WEBHOOK_SECRET`, `CASHFREE_ENVIRONMENT`,
`AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, `AWS_S3_BUCKET_NAME`, `AWS_S3_REGION`,
`AWS_S3_ENDPOINT_URL`, `CORS_ALLOWED_ORIGINS`, `CSRF_TRUSTED_ORIGINS`, `DEBUG_ENV`,
`ENVIRONMENT`, `DJANGO_SECURE_SSL_REDIRECT`, `DJANGO_HSTS_SECONDS`,
`DJANGO_LOG_FORMAT`, `LOG_LEVEL`, `BACKEND_URL`, `FRONTEND_URL`, `PORT`.

Must **never** be set in production, now enforced at boot: `TEST_PHONE_NUMBERS`,
`QA_ADMIN_BOOTSTRAP_PHONE`, `QA_ADMIN_BOOTSTRAP_CODE`, `QA_ADMIN_BOOTSTRAP_PASSWORD`.

Full matrix with per-variable classification: `production-environment-matrix.md`
(this commit).

## Secret rotation status

| Credential | Status |
|---|---|
| `POSTGRES_PASSWORD` (git history, `9af854c`) | **NOT ROTATED.** Compromised since 2026-05-07. |
| Google Maps API key (Android + iOS, one shared key) | **NOT ROTATED.** In every shipped APK; almost certainly unrestricted. |
| Railway QA secrets | Not sealed; readable by anyone with project access. |
| Production secrets | Do not exist yet. Create them **sealed**. |
| Firebase client config | Public by design. No rotation needed. |

No secret was rotated by me. Rotation touches live infrastructure and belongs to the
account owner.

## Mobile build status

| App | Tests | Analyze | APK build | Artifact |
|---|---|---|---|---|
| Rider | 68 pass, 7 skip | no errors/warnings, 32 infos | **Was impossible** (null cast in signing config). Fixed. **CI now green.** | `rider-debug-apk-34b3069`, 94,990,242 bytes |
| Driver | 83 pass | no issues | **Was impossible** (manifest did not parse). Fixed. **CI now green.** | `driver-debug-apk-bae0a3e`, 88,677,548 bytes |

**Both apps now produce an installable APK, for the first time.** Verified: run
`36026960814` (rider) and `36026805896` (driver), both `completed/success`, both with
a downloadable artifact.

These are debug-signed and are for install-and-drive testing, not for a store.
Neither app has been run on a physical device or emulator by me.

## Ops security status

Auth is sound in design: `admin_required` requires authenticated **and**
(`role == 'admin'` or `is_superuser`), applied to every view, with test coverage.
What is missing is everything around it: **no rate limiting, no MFA, no session
timeout review, no RBAC separation between an operator who answers support tickets
and one who approves payouts.** The console sits at the backend root.

## Cashfree blocker

Pinned three major versions behind (3.2.12 → 6.0.1). It blocks **all 14** dependency
vulnerabilities and it is the reason `transferId` idempotency is unverified. Do not
upgrade without the H6 suite. Do not invent a status endpoint.

## Financial invariant status

Holding: commission from the zone's `RateCard` not a global default; driver
settlement not spendable as rider credit; cash settlement writes one
`TransactionHistory` row; `credit_driver_wallet` idempotent on balance;
`TripSettlement` unique per trip+version; exactly one provider create call per
payout execution; refunds idempotent by key. **New:** an unknown payout outcome
holds the money instead of refunding it. `final_fare` remains NULL and unbilled.
**Not re-verified this run:** settlement concurrency, historical immutability, money
precision boundaries, a live cash ride.

## Redis failure status

Fixed: driver availability, trip-creation coherence, the payout-signal swallow.
Classified but unchanged: 50-odd other fail-open returns. **Unproven:** driver
exclusivity under concurrent acceptance with Redis unavailable (F2). SOS survives a
Redis outage as durable-but-unannounced.

## Celery durability status

`acks_late` + `reject_on_worker_lost` + prefetch 1 + time limits, all 13 tasks
covered, inventory guarded by test, both money sweeps asserted scheduled. **Beat is
still embedded via `-B`** and must be separated before a second worker exists. No
real worker-kill test was run.

## SOS status

Strong. Durable, deduplicated in PostgreSQL, ownership-enforced, coordinate-free
logs, operator-visible without Redis. One gap: **with the broker down nobody is
paged**, so an operator watching the console is a launch requirement.

## GPS / privacy status

The best-engineered subsystem reviewed. Bounded streams, consumer-group drain,
sampled points, idempotent writer, untrusted client clock separated from ours, no
serializer or route exposes the trail, 10 access-control tests. **Disabled by
default** — enabling it is a pilot decision. No retention policy, deliberately.

## Load-test evidence

**None. No load test was run this session.** No claim about concurrent rides,
dispatch latency, Redis throughput, Celery backlog or WebSocket command latency is
supported by evidence from this run. The existing `test_long_ride_soak.py` and
`test_lifecycle_under_gps_load.py` prove bounded growth under sustained telemetry for
a *single* ride, not concurrency.

## App distribution readiness

| Item | Rider | Driver |
|---|---|---|
| Package ID | `com.deeptrics.saaradhigo` | `com.saaradhigo.driver` — **two different orgs** |
| App label | SaaradhiGo | SaaradhiGo Drive |
| Branding | SaaradhiGo throughout | SaaradhiGo throughout |
| Build | fixed, CI verifying | fixed, CI verifying |
| Signing | debug only; no release keystore | debug only; no release keystore |
| Push | Firebase configured | **no `google-services.json`; no FCM token** |
| Privacy policy URL | not verified | not verified |
| Support URL | not verified | not verified |
| Account deletion | endpoints exist (`me/delete`, `me/export`) | — |

The package-ID split is a business decision about who owns the store listings and
must be settled before either app is submitted.

## India pilot configuration

A human must configure, before the first ride:

1. **Service zone** — pickup polygon for the pilot city (Hyderabad polygon exists in
   tests only).
2. **Vehicle categories** — which of bike/auto/car/car_xl/car_premium are live.
3. **`RateCard`** per zone and vehicle type — base, per-km, per-minute, minimum fare,
   commission. Without a card, quoting refuses (already tested).
4. **Surge bounds** — MVA-2020 caps surge; the value must be set deliberately.
5. **Support contacts** — the grievance and fraud mailboxes on
   `PLATFORM_CONTACT_DOMAIN` must **exist and be monitored**. Still unconfirmed.
6. **Driver onboarding** — who approves KYC, with what second factor.
7. **Payment methods** — cash only for the pilot; keep `WALLET_TOPUPS_ENABLED=False`.
8. **Cashfree** — sandbox vs production, and payouts explicitly off until H6 passes.
9. **S3** — private bucket, IAM user scoped to it, no public read.
10. **Notifications** — an FCM project and `google-services.json` for **both** apps.
11. **`GPS_TRAIL_ENABLED`** — decide whether the pilot needs route evidence.

### Production data bootstrap (idempotent, no fake users)

`TripStatus` rows (`requested`, `accepted`, `reached`, `in_progress`, `completed`,
`cancelled`) are already created by `get_or_create` at use sites. Needed as a
migration or a management command, run once and safe to re-run: vehicle types, the
pilot `ServiceZone`, its `RateCard`s, and platform settings. **No seeded users, no
seeded drivers.**

## Genuine launch blockers

1. **Production cannot boot and has no configuration.** Zero application variables.
2. **Production deploys from `dev`.** Repoint before configuring, or the first push
   after configuration ships whatever a developer committed.
3. **Two credentials compromised and unrotated** — the database password (4.5 months)
   and the shared Maps key.
4. **The console that approves KYC and releases payouts has no MFA and no rate
   limiting.**
5. **The driver app cannot receive push notifications.** No `google-services.json`,
   so no FCM token, so no background ride offers.
6. **No APK has been installed on a real handset.** Both builds were broken until
   today. Both now build green in CI with downloadable artifacts, so this is finally
   a task someone can do rather than a blocker.
7. **No end-to-end acceptance ride at the current commit.** Ack semantics, trip error
   handling, payout classification and OTP delivery all changed this run.
8. **An `unresolved` payout cannot be resolved.** The state can be entered and there
   is no console action to leave it.
9. **14 dependency vulnerabilities, all gated on a three-major Cashfree upgrade.**
10. **Backups unverified.** No evidence a PostgreSQL backup exists or has ever been
    restored.

## Technical debt

- Beat embedded in the worker (`-B`).
- One single-process test isolation failure, narrowed to a 2-file reproduction.
- ~50 classified-but-unchanged fail-open handlers.
- The Django console's `views.py` is ~6,500 lines.
- Next.js console reads page 1 only of a paginated endpoint.
- 7 rider test skips, 9 backend skips, all with explicit reasons.
- `PIIRedactionFilter`'s phone pattern is `+91`-only.
- `auto_cancel_trip` registers under its full dotted path unlike its siblings.
- Two consoles, no UI deprecation marker yet.

## Human actions required

Exact, in order. None can be done by me.

1. **Rotate the database password.** RDS and the Railway Postgres service. Then
   update the consuming configuration. Do **not** rewrite git history first.
2. **Rotate and split the Maps key.** Google Cloud Console → Credentials → create two
   keys → restrict the Android key to `com.deeptrics.saaradhigo` + release SHA-1, the
   iOS key to its bundle ID → replace in `AndroidManifest.xml` and `AppDelegate.swift`
   → delete the old key.
3. **Repoint production off `dev`.** Railway → project SaaradhiGo → environment
   `production` → service `SaaradhiGo-backend` → Settings → Source → change the
   branch trigger to a tag or disable automatic deploys.
4. **Resolve the staged `EnvironmentPatch`** pending since 2026-09-22 (Railway →
   production → review staged changes).
5. **Populate production variables, sealed**, from `production-environment-matrix.md`,
   with `TEST_PHONE_NUMBERS` and every `QA_ADMIN_BOOTSTRAP_*` absent. The boot guards
   will now tell you what is still missing.
6. **Create `google-services.json` for both apps** from an FCM project, and
   re-enable the `com.google.gms.google-services` plugin in the driver app's
   `build.gradle.kts`.
7. **Put the ops console behind an identity proxy with MFA**, or install
   `django-two-factor-auth`.
8. **Confirm the grievance and fraud mailboxes exist and are monitored.**
9. **Decide the store ownership** for the two package IDs.
10. **Either create `EC2_HOST_KEY` and restore the host, or delete the `deploy` job.**
    A permanently red pipeline is worse than no pipeline.
11. **Verify a PostgreSQL backup exists and restore it once** into a scratch database.

## Launch matrix

| Area | Status | Evidence |
|---|---|---|
| Rider build | **GREEN** | Was impossible (null cast in signing config). Fixed; CI run `36026960814` green with a 95 MB APK artifact. Debug-signed, never installed on a handset. |
| Rider core journey | **YELLOW** | 68 tests pass and the recovery/status-vocabulary defects are fixed, but C5 full-journey widget tests were not written. |
| Driver build | **GREEN** | Manifest did not parse, so no APK had ever existed. Fixed and locked by 5 tests; CI run `36026805896` green with an 89 MB APK artifact. |
| Driver core journey | **RED** | No push notifications at all (no `google-services.json`), so no background ride offer reaches a driver. |
| Ops | **YELLOW** | Canonical console decided, KYC route repaired and endpoint-verified; no operator has been told in writing which console to use. |
| KYC | **YELLOW** | `/driver/admin/` verified to exist and be protected; approve/reject flow not exercised end to end this run. |
| Ride | **GREEN** | 715 backend tests; trip creation coherence fixed; idempotency and transition tests pass in one process. |
| Dispatch | **YELLOW** | Wave dispatch, heartbeat gating and vehicle-type filtering are tested; behaviour with Redis unavailable mid-dispatch is unproven. |
| GPS | **GREEN (disabled)** | Bounded streams, consumer-group drain, idempotent writer, 10 access-control tests — and `GPS_TRAIL_ENABLED=False`, so nothing is being stored. |
| Reconnect | **GREEN** | Full reconnect state machine passes, including a completion whose ack was lost. |
| SOS | **YELLOW** | Durable, deduplicated, ownership-enforced, coordinate-free, operator-visible under a broker outage — but unannounced when Redis is down. |
| Support | **RED** | Not verified this run; ticket routing to operators untested. |
| Fare | **GREEN** | Server-computed, breakdown surfaced to the rider with surge disclosed, 17 widget tests. |
| Cash | **YELLOW** | Idempotent, gated on completion, one history row — but no live cash ride at this commit. |
| Online payment | **BLOCKED** | Cashfree three majors behind; webhook verification unproven against 6.x. |
| Settlement | **GREEN** | `TripSettlement` unique per trip+version, idempotent crediting, commission from the zone card. |
| Earnings | **YELLOW** | Reads settlement rows; not re-verified this run. |
| Payout | **RED** | An unknown outcome now correctly holds the money — and there is no way to resolve the resulting `unresolved` state. |
| Receipt | **YELLOW** | Issued by a Celery task, now durable against worker loss; retry semantics not exercised. |
| PostgreSQL | **GREEN** | Authoritative, row locks proven under contention, no pending migrations, 97 postgres tests pass together. |
| Redis | **YELLOW** | Three fail-open defects fixed; ~50 classified handlers unchanged; F2 unproven. |
| Celery | **YELLOW** | All 13 tasks durable against worker loss and guarded by test; beat still embedded via `-B`. |
| S3 | **YELLOW** | Presigned uploads tested; no production bucket or IAM policy exists. |
| Security | **RED** | No MFA and no rate limiting on the console that approves KYC and releases money. |
| Secrets | **RED** | Two compromised credentials unrotated; nothing sealed in either environment. |
| CI | **YELLOW** | Four repositories have CI and both apps now build — but the backend pipeline is permanently red on a broken EC2 deploy job. |
| Observability | **YELLOW** | Structured events exist for dispatch, ack timeout, socket loss, SOS, settlement and now payout-unresolved and autocancel-enqueue-failure; no dashboards or alerts. |
| Production deployment | **BLOCKED** | Cannot boot, no configuration, tracks `dev`. |
| Backups | **RED** | No evidence any backup exists or has ever been restored. |
| App distribution | **RED** | No release signing, no store accounts confirmed, package IDs owned by two different orgs. |
| India business/legal | **RED** | MVA-2020 entity name on receipts unconfirmed; grievance mailboxes unconfirmed; retention policy not approved. |

## Top 10 actions, risk ordered

1. Rotate the leaked **database password**.
2. Rotate and platform-restrict the **Maps key**.
3. **Repoint production off `dev`** — before configuring it.
4. **MFA on the ops console** before anyone approves a KYC or a payout.
5. **`google-services.json` for both apps**; re-enable the driver's plugin.
6. **Populate production variables, sealed**; let the boot guards check you.
7. **Install both APKs on real handsets** and drive one trip.
8. **Run the L1 acceptance ride at the current commit.**
9. **Verify and rehearse a PostgreSQL restore.**
10. **Build the console action that resolves an `unresolved` payout.**

## Next five engineering actions

Exactly five, in order:

1. **Add the `unresolved` payout resolution flow** to the Django console: show the
   `transferId`, require a human to record what Cashfree says, then move the row to
   `processed` or `failed` — and refund only on a confirmed `failed`. Right now a
   payout can enter a state it cannot leave.
2. **Run L1 end to end against QA at `7d609b1`** and capture trip id, settlement id,
   fare, commission, net, receipt status. Four subsystems changed this run.
3. **Prove F2**: driver exclusivity under concurrent acceptance **with Redis
   unavailable**, as a PostgreSQL contention test. The row lock is the last defence
   and it has never been tested in that condition.
4. **Separate Celery beat from the worker** and add the two-process topology to
   compose and the production plan.
5. **Write the Cashfree sandbox suite from H6** — starting with `transferId`
   idempotency, because every retry story and all 14 dependency CVEs sit behind it.

---

# FINAL EXECUTIVE TEST

## CAN WE ONBOARD 10 REAL DRIVERS?

**NOT YET.**

A driver needs an installable app, a KYC approval, and notifications. Until today no
driver APK could be built at all — the manifest did not parse. That is fixed and CI
now produces an 89 MB APK, but nothing has been installed on a handset. The app registers **no
FCM token**, so a driver whose app is backgrounded gets no ride offer. KYC approval
runs through a console with no MFA and no rate limiting. And onboarding starts with
an OTP whose delivery task, until this run, reported success without sending.

## CAN WE SERVE 100 CONTROLLED CASH RIDES?

**NOT YET.**

The ride path itself is the strongest part of the system: 715 backend tests, all 97
concurrency tests passing together, idempotent booking, a proven reconnect state
machine, fares computed server-side and now shown to the rider with surge disclosed.
But there is **nowhere to run them** — production cannot boot and has no
configuration — and **no acceptance ride has been run at the current commit**, after
a run that changed Celery ack semantics, trip-creation error handling and OTP
delivery. Cash settlement is sound in test and unverified live today.

## CAN WE ACCEPT REAL ONLINE PAYMENTS?

**NOT YET — and this one is BLOCKED, not merely incomplete.**

`cashfree-pg` is three major versions behind. Webhook signature verification against
6.x payloads is unproven, `transferId` idempotency is unproven, and all 14 known
dependency vulnerabilities sit behind that single pin. The H6 sandbox suite must pass
first. `final_fare` billing remains off and should stay off.

## CAN WE RELEASE RIDER/DRIVER APPS?

**NOT YET.**

Both apps were unbuildable until today, for two unrelated reasons. **Both now build
green in CI and produce downloadable APKs** (95 MB and 89 MB), which is a real change
in position. Beyond that: no release signing keystore exists, the
two package IDs belong to **two different organisations** with no decision on who
owns the store listings, the driver app has no push configuration, and privacy-policy
and support URLs are unverified. Debug APKs for technical testing are within reach
this week. Store submission is not.

## CAN AN OPERATOR RUN THE PILOT WITHOUT A DEVELOPER?

**NOT YET.**

The Django console is genuinely capable — 19 pages covering KYC, trips, payouts, SOS,
zones and refunds — and it is now the declared canonical one. But an operator cannot
resolve an `unresolved` payout, because that action does not exist. They have not
been told in writing which of the two consoles to use. There are no dashboards or
alerts, so the first sign of trouble is a phone call. And an SOS raised while Redis
is down is recorded but **not announced**, which means the pilot depends on an
operator watching a screen rather than on being paged.

---

**What changed in the balance of risk tonight.** Every previously-unreviewed
subsystem — driver app, Celery, Redis failure behaviour, payouts, SOS, dependency
security, GPS — has been examined, and the two genuinely dangerous money and safety
behaviours found in them are fixed. The remaining blockers are now almost entirely
**configuration, credentials, and verification that has not happened**, plus one
product gap (resolving an ambiguous payout). That is a materially better position
than a design flaw. It is still not readiness, and 715 passing tests do not make it
so.

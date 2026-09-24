# India launch proof report

2026-09-24. Integration-proof run. The priority was to prove the system as it
stands against the current QA revision before changing anything, and that is what
happened.

**Scope honesty up front.** Of the 70 phases requested, **Phases 1–6 were completed
with evidence**, plus parts of 25, 34 and 61 that fell out of them. Phases 7–24 and
26–70 were **not executed**. The report says so per phase rather than implying
coverage. No secret value appears anywhere below.

---

## Exact repository HEADs

Recorded at the start, all clean and pushed:

| Repository | Branch | HEAD at freeze |
|---|---|---|
| SaaradhiGo-backend | `dev` | `7d609b1dd36e3f3048c59519f166f4fa45d1bca7` |
| SaaradhiGo-mobile (rider) | `dev` | `34b3069800ad657dd87bb36ea7e70d92851bc425` |
| SaaradhiGo-driver | `feat/driver-app-rebuild` | `bae0a3ea83f08ecc932ee10b1a5f14e91943b63b` |
| SaaradhiGo-web (ops) | `develop` | `a7989824cc4585adcf725a3e60b42d9414f532bd` |
| SaaradhiGo-docs | `docs/india-launch-readiness` | `7c7ea281b8a38cabba95b96ac349be9cfff1803d` |

Backend HEAD now `7747cd9` — two commits added, **both non-runtime**
(`tests/test_driver_exclusivity_redis_down.py` and `qa/acceptance_ride.py`).
`git diff --name-only 7d609b1..HEAD -- servers/ base/` is empty, so the runtime
code under test is byte-identical to the revision the acceptance ride ran against.
That is why no second acceptance ride was required.

## QA revision

Verified through the revision endpoint built for exactly this purpose, **not** by a
200 from `/healthz`:

```
GET /version
{"service":"saaradhigo-backend","environment":"qa",
 "revision":"7d609b1dd36e3f3048c59519f166f4fa45d1bca7",
 "revision_source":"RAILWAY_GIT_COMMIT_SHA","revision_short":"7d609b1",
 "build_time":"unknown"}

GET /healthz  ->  {"status":"ok","db":"ok","cache":"ok", ...same revision...}
```

`revision_source: RAILWAY_GIT_COMMIT_SHA` is the platform-injected value, so it
updates itself. **Phase 34 is proven in production use**: a deploy verifier can
assert the expected revision. This directly closes the failure mode that produced a
false defect report two runs ago.

Railway QA deployment `3d5ee0d8`, `commitHash 7d609b1`, `SUCCESS` at 16:28:33Z.
Migrations run as `preDeployCommand: python manage.py migrate --noinput`, so the
schema is applied by the same successful deploy.

Service state at freeze:

| Service | State |
|---|---|
| `backend` (Daphne) | live, `7d609b1`, `db: ok`, `cache: ok` |
| `celery` | live, `7d609b1`, `celery -A base worker -B -l INFO --concurrency 2` — worker **with embedded beat** |
| `Postgres`, `Redis` | live |
| `alluring-happiness` | **crashed**, `isDeleted: true`, `state: staged-delete`, **zero variables** |

`alluring-happiness` is a stray service whose deletion was staged on 2026-09-23 and
never committed. It crash-loops on the `ALLOWED_HOSTS` guard on every `dev` deploy
because it has no configuration. Harmless to the product, but it is why QA
permanently reports "1 service crashed" — alert fatigue on the environment you most
need to trust. Committing that staged deletion is a one-click human action.

**Freeze.** No deployment occurred between the freeze and the end of the ride. The
mechanism was simply not pushing to `dev`; the two commits above were pushed after
the ride completed.

---

## Acceptance Ride #1

**29 of 30 stages PASS.** Trip **45**, revision `7d609b1`, real QA PostgreSQL,
Redis, Daphne/Channels and Celery. No mocks on the main path.

```
PASS RIDER_AUTH            http=200
PASS DRIVER_AUTH           http=200
PASS FARE_QUOTE            quote=135.22  components=base_fare,distance_fare,
                           min_fare_applied,surge_multiplier,time_fare
PASS DRIVER_ONLINE         location pinged, geo index settled
PASS RIDE_REQUEST          trip_id=45
PASS DISPATCH              drivers_notified=1
PASS DRIVER_OFFER          trip_id=45
PASS ACCEPT                ack=committed  trip_status=accepted
PASS RIDER_ASSIGNMENT      rider notified of driver
PASS DRIVER_REACHED        ack=committed  trip_status=reached
PASS OTP                   obtained via the rider channel (never printed)
PASS START                 ack=committed  trip_status=in_progress
PASS GPS                   36 frames over 184 s
PASS RIDER_LIVE_STATE      36 driver position updates seen by the rider
PASS SOS                   http=201  sos_id=1  status=open
PASS SOS_DURABLE_RECORD    http=200  repeat_of=1  (second press collapsed)
PASS COMPLETE              ack=committed  trip_status=completed
PASS COMMAND_ACK           correlated ack returned committed
PASS CASH_CONFIRM          ack=committed  trip_status=completed
FAIL FARE_SNAPSHOT         harness read the wrong endpoint -- see below
PASS FINAL_FARE_IS_NULL    final_fare absent/None
PASS DRIVER_EARNINGS       http=200
PASS WALLET_TRANSACTION    http=200
PASS RIDER_HISTORY         trip 45 present
PASS OPS_VISIBILITY        http=401 (exists, protected) -- see the boundary below
```

### Status transitions and command identity

| Command | `command_id` | ack | resulting `trip_status` |
|---|---|---|---|
| accept | `accept-aeaa1f8b21` | **committed** | accepted |
| reached | `reached-05723a57de` | **committed** | reached |
| start | `start-a23f8970c1` | **committed** | in_progress |
| complete | `complete-39afcf25f9` | **committed** | completed |
| confirm_cash | `confirm_cash-713d88b14d` | **committed** | completed |

Every lifecycle command returned a correlated `committed` ack with the correct
resulting status, over real infrastructure. The ACK protocol works.

### Timestamps (from the trip record)

```
requested_at  2026-09-24T22:23:56.930+05:30
accepted_at   2026-09-24T22:23:57.997+05:30
started_at    2026-09-24T22:24:42.278+05:30
completed_at  2026-09-24T22:27:46.216+05:30
ride duration 184 s
```

### GPS and actual metrics

- **36 frames sent**, 5 s apart, ~78 m per step.
- **36 relayed live to the rider** — the live path is complete.
- **0 durable points stored.** `actual_distance_km: None`,
  `actual_duration_min: None`, still None eight minutes later.

Cause confirmed in code, not guessed: `persist_location_trail` is a no-op while
`GPS_TRAIL_ENABLED` is False (its default), so there is nothing for
`compute_trip_actuals` to compute from. **There is currently no durable route
evidence for a dispute, an SOS investigation or an insurance claim.** That is a
pilot configuration decision, not a defect — but it must be a decision.

### The one FAIL was mine

`FARE_SNAPSHOT` failed because the harness called `/ride/trip/45/details/`, which
is `trip_driver_details` (driver name, phone, rating, vehicle) and returned
`source: cache` with seven fields. The fare snapshot lives on `/ride/trip/45/`,
which `trip_detail` serves **from PostgreSQL always** — a Redis fast path was
deliberately removed from it earlier precisely because it authorised against cached
ids and returned a different field set. Read correctly:

```
estimated_fare   135.22
fare_breakdown   base_fare 60.00  distance_fare 55.59  time_fare 19.62
                 surge_multiplier 1.00  total_fare 135.22
final_fare       None
payment_status   completed
status           completed
```

Harness corrected and committed.

## Acceptance Ride #2 / final

**Not run, and not required.** Phase 67 asks for a final ride after all
runtime-affecting merges. The only two commits after Ride #1 are a test file and a
QA script; `git diff 7d609b1..HEAD -- servers/ base/` is empty. The runtime under
test is unchanged, so re-running would re-prove the same binary.

If any runtime change is merged, `qa/acceptance_ride.py` must be run again before
release. That is now a one-command gate rather than a manual reconstruction.

---

## Financial reconciliation

Phase 5, computed from the immutable trip evidence rather than from today's
RateCard:

```
gross (charged)        135.22
commission              27.04     = 20.00 % of gross
driver net             108.18
gross - commission     108.18     ->  matches net exactly
quote                  135.22     ->  equals the charged amount
final_fare             None       ->  metered billing not authoritative
payment_method         cash
cash_in_hand           True
earnings rows for 45   1
```

Independently verified across the whole earnings feed: **no trip has more than one
earnings row.** So the retried `complete` and `confirm_cash` produced no duplicate
commission anywhere, not just on trip 45.

### Phase 4 — retry equivalence

```
retry complete      ack=already_done   trip_status=completed
retry confirm_cash  ack=committed      trip_status=completed
retry receipt       http=200
estimated_fare unchanged      True
final_fare still null         True
duplicate wallet credit       none
earnings rows for trip 45     1
```

One completed trip, one financial effect, one commission. `already_done` is exactly
the right answer to a repeated terminal command.

### A real finding: the breakdown does not sum to the total

```
base_fare 60.00 + distance_fare 55.59 + time_fare 19.62 = 135.21
total_fare                                              = 135.22
```

One paisa. The components are each rounded to two decimals and the total is rounded
from unrounded inputs, so `sum(round(x)) != round(sum(x))`. A rider who adds up the
itemised breakdown in the fare-transparency panel gets a different number from the
amount charged.

Not fixed tonight, deliberately. `total_fare` equals what is charged, so changing it
would be a money change; the correct fix allocates the rounding remainder into one
component, which is fare code and belongs with a test and a review rather than at
the end of a proof run. **Recorded as a defect, severity low in amount and
non-trivial in trust.**

---

## Redis-down exclusivity

**Phase 6, the mandatory one: PROVEN. 5/5 tests pass** against PostgreSQL 15 with
real threads.

`tests/test_driver_exclusivity_redis_down.py`. Driver D available, riders R1 and R2,
two concurrent acceptances, Redis genuinely unreachable — a `_DeadRedis` whose every
attribute returns a callable that raises, deliberately not a `Mock`, because a Mock
would silently succeed and prove the opposite.

| Assertion | Result |
|---|---|
| Exactly one trip holds the driver | **PASS** — also confirmed by an independent count of the driver's active trips |
| The loser is refused deterministically | **PASS** — never told it succeeded |
| The second transaction really blocked in PostgreSQL | **PASS** — duration ≥ 60 % of the 1.5 s lock hold, so a green result cannot come from scheduling luck |
| No trip left half-assigned | **PASS** — loser stays cleanly `requested` and unassigned |
| Redis really was unavailable | **PASS** — negative control; `get_driver_active_trip` raises rather than guessing |

So PostgreSQL alone is sufficient: `driver_active_trip_ids` queries the Trip table
and `_accept_trip` takes `select_for_update` on the Driver row. The Redis cache is an
optimisation, not the guard.

### A bug in my own test, worth recording

My first version applied `mock.patch.object(rc, 'redis_client', ...)` **inside each
worker thread**. `mock.patch` on a module global is not thread-safe: with two threads
in overlapping context managers, the second saves the *first thread's stub* as "the
original" and restores the stub on exit, leaving the module permanently patched.
`_DeadRedis` leaked into later tests and failed **five unrelated PostgreSQL tests**.
Fixed by patching once, in the calling thread, spanning both workers. Full postgres
suite now **75 passed, 0 failed** in one process.

## Redis outage behaviour

Partially covered, and honestly so.

- **Trip creation with the broker down** — covered by
  `test_trip_creation_broker_coherence.py`: a committed trip is no longer reported as
  a failed booking. Residual risk documented in the code: such a trip has no
  scheduled timeout.
- **Acceptance with Redis down** — proven above.
- **SOS with the broker down** — proven durable and operator-visible.
- **Phase 7 (outage during `accepted`, during `in_progress`, immediately before
  `complete`) was NOT executed.** Reconnect-after-recovery reconstructing
  PostgreSQL truth is covered by the existing reconnect state machine tests for
  socket loss, not for Redis loss.

## Celery worker-loss behaviour

Not re-tested this run. Standing position from the previous run, unchanged:
`task_acks_late` and `task_reject_on_worker_lost` are on globally, prefetch is 1, all
13 registered tasks are covered by an inventory guard, and both money sweeps are
asserted scheduled. **Phases 14 and 15 — an actual kill-the-worker matrix and
per-task duplicate-execution proofs — were NOT executed.** Durability is asserted
from configuration, which is weaker than an interruption test.

**Beat is still embedded in the worker** (`-B`), confirmed live in QA this run. Safe
only while exactly one worker runs.

## Driver FCM

**Phases 9, 10, 11 not executed.** Unchanged and launch-blocking: the driver app
depends on `firebase_messaging`, the `com.google.gms.google-services` plugin is
commented out because `google-services.json` is absent, `Firebase.initializeApp()`
throws, `PushService` catches it and sets `_ready = false`, and the app registers no
token. A backgrounded driver receives no ride offer.

## Rider mobile

Untouched this run. 68 tests passing, 7 skipped, analyze clean, APK builds green in
CI (95 MB artifact). The fare-transparency panel is wired and shows the breakdown
whose one-paisa inconsistency is described above. **Phases 17, 18, 20, 50, 56, 57 not
executed.**

## Driver mobile

Untouched this run beyond confirming the baseline: 83 tests passing, analyze clean,
APK builds green in CI (89 MB artifact). **Phases 19, 20, 51 not executed.**

## Ops

Not advanced. The Django console remains canonical. The KYC route repair is verified
at the route level (401 vs 404).

**A hard boundary was hit: there is no QA operator account.** The three
`QA_ADMIN_BOOTSTRAP_*` variables exist in QA and are read by **nothing in the
codebase** — the only references are in the production guard I added last run. So
they are vestigial config, one of them named `_PASSWORD`, and there is no mechanism
to create an operator.

Consequence: **authenticated operator visibility is unproven, and Phases 21–24, 26
and 60 could not be executed.** I did not create an operator account, because
minting a credential in a shared environment is a human decision, not a test step.

## MFA / rate limiting

**Phases 21 and 22 not executed.** Unchanged: the console that approves KYC and
releases payouts has neither. Recommended approach remains an identity proxy in front
of the console so MFA is enforced before a request reaches Django.

## SOS

Proven live on the acceptance ride: `sos_id=1`, `status=open`, HTTP 201, and a second
press returned HTTP 200 with `repeat_of=1` — deduplicated, not spammed. Combined with
the six broker-outage tests from the previous run (durable, operator-visible,
coordinate-free logs), SOS is the best-evidenced safety surface in the product.

**Phase 13's remaining item — that retrying the notification creates no duplicate SOS
record — is covered by the dedup proof.** Phase 69 was not run separately because
SOS was exercised inside Ride #1.

## Support

**Phase 48 not executed.**

## Payout ambiguity

**Phase 25 partially covered by existing tests, not re-run live.** The standing
position: a post-send network failure or a 5xx raises `PayoutDispatchUnknown`, the
withdrawal becomes `unresolved`, the wallet is **not** refunded, no provider
reference is recorded, and the row cannot be re-dispatched. A pre-send validation
failure or a 4xx is a definite rejection and still refunds. 11 tests including both
negative controls.

**Phase 26 not executed and still open:** an `unresolved` payout can be entered and
**cannot be left** — there is no operator action to resolve it. That is a product gap,
not a missing test.

Observed incidentally on the acceptance driver: `wallet_balance: -705.58`,
`available_balance: -705.58`. Economically correct for cash rides — the driver
collected cash and owes commission — but a driver shown a negative "available
balance" next to `minimum_withdrawal: 500.0` will not read it that way.

## Cashfree blocker

**Phase 27 not executed beyond the plan already written.** `cashfree-pg` pinned at
3.2.12 against a latest of 6.0.1. The sandbox suite specified in the previous report
stands, and item 2 — the same `transferId` replayed must not create a second transfer
— remains the linchpin for every retry story.

## Receipt

Receipt resend accepted (HTTP 200) and economics unchanged afterwards. **Phase 49's
failure cases (S3 down, email down, duplicate task) were NOT executed.**

## GPS

Live relay proven (36/36 frames reached the rider). Durable trail **disabled**, so
zero points stored and no actual metrics. **Phases 39 and 40 not executed this run**;
Phase 40's ground is already covered by 10 existing access-control tests including
cross-rider and cross-driver IDOR.

## GPS retention

**Phase 38 not executed.** The storage model from the previous run stands:
≈250–400 bytes per point including index overhead, ≈200–300 points per urban ride,
so 2–4 GB/year at 100 rides/day and 180–440 GB/year at 10,000.

## Security / secrets

**Phase 30 not re-run.** The previous scan's classification stands: two active
credentials requiring rotation, everything else test/fake or public-by-design.

New this run: `QA_ADMIN_BOOTSTRAP_PHONE`, `QA_ADMIN_BOOTSTRAP_CODE` and
`QA_ADMIN_BOOTSTRAP_PASSWORD` are **dead configuration** — nothing reads them. They
should be removed from QA rather than left as three variables that look like live
admin credentials.

## Database credential rotation status

**NOT ROTATED.** `POSTGRES_PASSWORD`, committed in `9af854c` on 2026-05-07, reachable
from `dev` and five other branches, therefore in every clone. Exposed ~4.5 months.
Value not reproduced. **Phase 28's "is it still active" check was not performed**,
because confirming it means attempting a connection with a known-leaked credential
against a live database, which is a decision for the owner and not something to do
unasked.

## Maps credential rotation status

**NOT ROTATED.** One key shared by `AndroidManifest.xml` and `AppDelegate.swift`, so
it cannot be platform-restricted on both and is almost certainly unrestricted and
billable by anyone who extracts it from the APK. **Phase 29 remediation steps stand
from the previous report; nothing was changed.**

## Dependency vulnerabilities

**Phase 59 not re-run.** Standing proof: 14 advisories across `urllib3 2.0.7` (6
distinct) and `sentry-sdk 1.32.0` (1), all blocked by `cashfree-pg==3.2.12` declaring
`urllib3 <2.1.0` and `sentry-sdk <1.33.0`. `botocore` and `requests` both permit
`urllib3 <3`, so they are not the constraint. **Independently fixable: zero.**

## Load-test evidence

**None. Phases 41 and 42 were not executed.** No concurrency evidence exists. The
acceptance ride was a single ride. Nothing in this report supports any claim about 10,
25 or 50 simultaneous rides.

## PostgreSQL performance

**Phase 43 not executed.** No query plans were captured.

## CI / build gates

Unchanged and currently complete for the four repositories: backend (tests, lint,
postgres integration), rider (analyze, tests, **APK build**), driver (analyze, tests,
**APK build**), ops (lint, typecheck, build). **Phase 58's "no launch-critical repo
may have only lint/test without build" is satisfied.**

The backend pipeline remains **permanently red** on the EC2 `deploy` job while
`test`, `lint` and `postgres-concurrency` are green.

## Production release architecture

**Phases 31, 32, 33 not executed this run.** The model proposed previously stands:
`dev` → QA automatically; production from an **immutable tag**, promoted by a human.
Production still tracks `dev`, which remains blocker #2.

The production boot rehearsal from the previous run still holds — 15 subprocess tests
proving production refuses to start on DEBUG=True, missing `DJANGO_SECRET_KEY`,
missing `ALLOWED_HOSTS`, `TEST_PHONE_NUMBERS`, any `QA_ADMIN_BOOTSTRAP_*`, or QA URLs,
with a negative control proving a correct configuration does boot.

## Backup readiness

**Phase 35 not executed. Production remains RED on recovery.** No evidence any
PostgreSQL backup exists or has ever been restored.

## Rollback rehearsal

**Phase 65 not executed.**

## First 10 drivers

**Phase 63 not executed** — `runbooks/first-10-drivers.md` was not written. Writing an
operator runbook while no operator account exists would be fiction.

## First 100 rides

**Phase 64 not executed** — `runbooks/first-100-rides.md` not written, for the same
reason.

---

## Genuine launch blockers

Ordered by risk. Items 1–3 are new or newly demonstrated this run.

1. **An abandoned in-progress ride strands the driver indefinitely.** *New, and
   demonstrated.* Trip 42 reached `in_progress` and its clients died. The rider
   cannot cancel a started ride (correct policy). `auto_cancel_trip` only covers
   *unaccepted* trips, so there is no timeout. `/ride/active/` returned trip 42
   forever, and the driver's Redis active-trip key never cleared — the next dispatch
   reported `drivers_notified=0` and a second and third booking found no driver.
   Recovery required a **developer** opening the driver's trip socket and issuing
   `complete`. For a ten-driver pilot, one crashed app removes 10 % of supply until
   an engineer intervenes.
2. **No QA operator account exists**, and the three `QA_ADMIN_BOOTSTRAP_*` variables
   that look like they would create one are read by nothing. Authenticated operator
   visibility is therefore unproven, and the operator drill cannot be rehearsed.
3. **No load evidence of any kind.** Not a single concurrency measurement exists.
4. **Production cannot boot and has no configuration** (8 Railway-injected variables
   against QA's 30).
5. **Production deploys from `dev`.** Repoint before configuring.
6. **Database password compromised and unrotated**, ~4.5 months.
7. **Maps key compromised, shared across platforms, unrotated.**
8. **Ops console has no MFA and no rate limiting.**
9. **The driver app registers no FCM token** — no background ride offers.
10. **An `unresolved` payout cannot be resolved** by anyone.
11. **Backups unverified.**
12. **14 dependency advisories**, all behind the Cashfree pin.

## Technical debt

- Fare breakdown components sum to one paisa less than the total charged.
- `alluring-happiness`: crashed stray service, staged-delete never committed, source
  of a permanent false "crashed" signal in QA.
- `QA_ADMIN_BOOTSTRAP_*`: three dead variables that read as live admin credentials.
- Beat embedded in the worker via `-B`.
- Driver "available balance" shows negative for cash-heavy drivers with no
  explanation.
- One postgres test still needs its own process (`test_lifecycle_under_gps_load.py`
  after `test_driver_active_trip_contention.py`).
- `GPS_TRAIL_ENABLED=False` means no durable route evidence — a decision, not a bug,
  but currently undecided.

---

## Launch decision matrix

| Item | Status | Evidence |
|---|---|---|
| Rider APK | **GREEN** | CI run `36026960814` green, 94,990,242-byte artifact. |
| Driver APK | **GREEN** | CI run `36026805896` green, 88,677,548-byte artifact. |
| Rider core journey | **GREEN** | Ride #1: quote, request, assignment, live position, history all passed on real QA. |
| Driver core journey | **YELLOW** | Accept→reached→start→complete→cash all `committed`, but no background offer and nothing device-tested. |
| Driver background offer | **RED** | No `google-services.json`; app registers no FCM token. |
| Auth | **GREEN** | Rider and driver login 200 on QA in every run. |
| OTP | **YELLOW** | Delivery retries proven by 14 unit tests; no live SMS provider proof (Phase 12 not executed). |
| KYC | **YELLOW** | Route verified 401-not-404; approve/reject never exercised, no operator account. |
| Ops | **RED** | No QA operator account exists; authenticated visibility unproven. |
| MFA | **RED** | Not implemented. |
| Ride request | **GREEN** | Trip 45 created with a client-generated idempotency key. |
| Dispatch | **GREEN** | `drivers_notified=1`, offer delivered to the driver socket. |
| Driver exclusivity | **GREEN** | 5/5 PostgreSQL concurrency tests with Redis genuinely dead; second transaction provably blocked. |
| Lifecycle | **GREEN** | All five commands returned correlated `committed` acks with correct statuses. |
| Reconnect | **YELLOW** | Existing state-machine tests pass; no reconnect was exercised during Ride #1 (Phase 70 not run). |
| GPS | **YELLOW** | 36/36 frames relayed live; **zero durable points**, actual metrics None, trail disabled. |
| SOS | **GREEN** | Live: `sos_id=1` open, second press returned `repeat_of=1`; plus 6 broker-outage tests. |
| Support | **RED** | Not verified at all. |
| Fare | **YELLOW** | Snapshot immutable and complete; components sum one paisa short of the total. |
| Cash | **GREEN** | `135.22 − 27.04 = 108.18` exact, one earnings row, retries `already_done`. |
| Online payment | **BLOCKED** | Cashfree three majors behind; provider contract unverified. |
| Settlement | **GREEN** | Exactly one earnings row for trip 45; no trip in the feed has more than one. |
| Wallet | **YELLOW** | Balance endpoint works; negative available balance is correct but unexplained to the driver. |
| Earnings | **GREEN** | Commission 20.00 %, net reconciles from immutable trip evidence. |
| Payout | **RED** | Ambiguity handled correctly; `unresolved` state cannot be resolved by anyone. |
| Receipt | **YELLOW** | Resend 200 and economics unchanged; failure paths untested. |
| PostgreSQL | **GREEN** | 715 tests on SQLite, 75 in one process on PG 15, migrations applied by pre-deploy. |
| Redis | **YELLOW** | Exclusivity survives loss; full outage matrix (Phase 7) not executed. |
| Celery | **YELLOW** | Worker and beat live on the tested revision; no kill-the-worker matrix run. |
| S3 | **YELLOW** | Configured in QA; no production bucket or IAM policy; KYC file security untested. |
| Secrets | **RED** | Two compromised credentials unrotated; nothing sealed. |
| Maps | **RED** | One key shared Android/iOS, unrestricted, unrotated. |
| Dependencies | **BLOCKED** | 14 advisories, zero independently fixable. |
| CI | **YELLOW** | All four repos have build gates; backend pipeline permanently red on EC2 deploy. |
| Load | **RED** | No measurement of any kind exists. |
| Observability | **YELLOW** | Structured events exist including `payout_unresolved` and `trip_autocancel_enqueue_failed`; no dashboards or alerts. |
| Backups | **RED** | Unknown whether any backup exists. |
| Rollback | **RED** | Never rehearsed. |
| Production config | **RED** | 8 variables, all Railway-injected. |
| Production deployment | **BLOCKED** | Cannot boot; tracks `dev`. |
| App distribution | **RED** | No release signing; two package IDs owned by different orgs. |
| India legal/business | **RED** | Receipt legal entity unconfirmed; grievance mailboxes unconfirmed. |

---

## Final answers

### Can we technically onboard the first 10 drivers?

**NOT YET.**

Minimum remaining condition: a working operator account plus MFA, so KYC can be
approved by a person rather than an engineer; `google-services.json` for the driver
app so a driver receives offers when the app is backgrounded; and one APK installed
and driven on a real handset. The build blockers are gone — both APKs exist as CI
artifacts — so this is now onboarding work, not engineering.

### Can we technically serve 100 controlled CASH rides?

**NOT YET**, but closer than anything else on this list.

The ride works end to end on real infrastructure and the money reconciles exactly.
Two conditions remain. First, **stuck-ride recovery**: an abandoned in-progress trip
currently removes a driver from supply until a developer intervenes, and across 100
rides that will happen. Second, **somewhere to run them** — production cannot boot.

### Can we enable online payments?

**NOT YET.**

Minimum remaining condition: the Cashfree sandbox suite passing, beginning with
`transferId` idempotency and webhook signature verification against 6.x. Until then
the provider contract is assumed, and 14 dependency advisories sit behind the same
pin.

### Can we process real driver payouts automatically?

**NOT YET**, and this one should stay NOT YET deliberately.

Ambiguity is now handled correctly — an unknown outcome holds the money rather than
refunding it. But the `unresolved` state has no exit, and automatic resolution
requires a provider status call that has never been verified. Minimum remaining
condition: the operator resolution flow, then the sandbox proof. Manual payouts with
a human checking Cashfree are the correct posture for a pilot.

### Can operators run the pilot without developer intervention?

**NOT YET**, and this run produced direct evidence rather than an opinion: recovering
one stuck trip required a developer opening a WebSocket as the driver, and there is
no operator account in QA to attempt it any other way.

Minimum remaining condition: an operator account, MFA, a stuck-ride action, and a
payout-resolution action.

### Can we deploy production?

**NOT YET.**

Minimum remaining condition, in order: rotate the leaked database password; repoint
the production service off `dev`; populate the production variables sealed; confirm a
backup exists and restore it once. The boot guards will then report anything still
missing, which is what they are for.

---

## Human actions — next 5

1. **Rotate the leaked PostgreSQL password** on RDS and the Railway Postgres
   service, then update the consuming configuration. Do not rewrite git history
   first; rewriting does not end the exposure and rotation does.
2. **Repoint the production Railway service off `dev`** (Railway → SaaradhiGo →
   production → `SaaradhiGo-backend` → Settings → Source) to a tag or manual
   trigger. Do this **before** populating any variable.
3. **Create a QA operator account and put the console behind MFA.** Until one
   exists, no operator procedure can be written or rehearsed, and four phases of
   this run stayed blocked on it.
4. **Create an FCM project and `google-services.json` for both apps**, then
   re-enable `com.google.gms.google-services` in the driver app's
   `build.gradle.kts`.
5. **Rotate and split the Maps key** — one Android key restricted to the package
   plus release SHA-1, one iOS key restricted to the bundle ID — and commit the
   staged deletion of the `alluring-happiness` service while in the Railway console.

## Engineering actions — next 5

1. **Stuck-ride recovery.** A timeout or sweep for trips left in `in_progress` past a
   bound, plus an operator action to force-complete or cancel one, which must clear
   the driver's active-trip key. Demonstrated need, not speculative.
2. **The `unresolved` payout resolution flow** in the Django console: show the
   `transferId`, require a human to record what Cashfree reports, then move the row
   to `processed` or `failed` — refunding only on a confirmed `failed`.
3. **The load harness (Phases 41–42)** at 10, then 25, then 50 concurrent rides with
   realistic GPS, against isolated infrastructure. There is currently no concurrency
   evidence at all.
4. **The Redis outage matrix (Phase 7)**: outage during `accepted`, during
   `in_progress`, and immediately before `complete`, asserting PostgreSQL stays
   coherent and completion cannot double.
5. **Fix the fare breakdown rounding** so the components sum to the amount charged,
   with a test — changing the decomposition, never the total.

---

## Core success condition

What is now evidenced end to end, on real QA infrastructure at a known revision:

```
BUILD        both APKs green in CI with artifacts
DEPLOY       QA on 7d609b1, verified by revision, not by HTTP 200
AUTHENTICATE rider and driver login
ONBOARD      (NOT proven -- no operator account to approve KYC)
REQUEST      trip 45 with a client idempotency key
DISPATCH     drivers_notified=1
ACCEPT       committed
REACH        committed
OTP          obtained via the rider channel
START        committed
GPS          36/36 frames relayed live  (0 durable points -- trail disabled)
SOS          sos_id=1 open, second press deduplicated
COMPLETE     committed
CASH         135.22 - 27.04 = 108.18, exact
SETTLE       exactly one earnings row, no duplicates anywhere
RECEIPT      resend 200, economics unchanged
HISTORY      trip present for the rider
EARNINGS     commission 20.00%, net reconciles
OPERATE      (NOT proven -- no operator account)
RECOVER      driver exclusivity survives Redis death, 5/5
```

Eighteen of twenty links are evidenced. The two that are not — **ONBOARD** and
**OPERATE** — are blocked on the same missing thing: an operator account and the
authority to act through a console rather than a shell. That, plus the stuck-ride
gap this run demonstrated, is what stands between a system that works and a pilot a
business can run.

# India pilot operations closure report

2026-09-24. This run began after the proof run and had one stated Priority 1: the
stranded in-progress ride that removed a driver from supply permanently and needed
an engineer to recover.

**Scope honesty.** Phases 7–22 were completed with evidence, plus 32/33's inventory
guard, 47's predecessor work and the final acceptance ride. Phases 23–31 and 34–70
were **not executed**, and each says so below rather than being implied. No secret
value appears anywhere.

---

## Exact HEADs

| Repository | Branch | HEAD |
|---|---|---|
| SaaradhiGo-backend | `dev` | `77bed61` (was `34328ef` at the acceptance ride; the only change after it is the estimate-fare view fix described in the appendix) |
| SaaradhiGo-mobile (rider) | `dev` | `719736a7e0ba21af0e8058b4122a605018cf819b` |
| SaaradhiGo-driver | `feat/driver-app-rebuild` | `bae0a3ea83f08ecc932ee10b1a5f14e91943b63b` |
| SaaradhiGo-web (ops) | `develop` | `a7989824cc4585adcf725a3e60b42d9414f532bd` |
| SaaradhiGo-docs | `docs/india-launch-readiness` | this commit |

Starting state was `7747cd9` / `34b3069`, all clean and pushed.

## QA revision

Baseline at freeze verified through `/version`, not a 200 from `/healthz`:

```
revision        7d609b1... -> 7747cd9... (start of run)
revision_source RAILWAY_GIT_COMMIT_SHA
environment     qa
healthz         status ok, db ok, cache ok
```

Services at freeze: `backend` live, `celery` live running
`celery -A base worker -B -l INFO --concurrency 2` (worker **with embedded beat**),
Postgres and Redis live. Migrations apply via
`preDeployCommand: python manage.py migrate --noinput`.

One finding recorded at baseline: the QA service `alluring-happiness` is
`isDeleted: true`, `state: staged-delete`, **zero variables**, and crash-loops on the
`ALLOWED_HOSTS` guard with every `dev` deploy. Its deletion was staged on 2026-09-23
and never committed, so QA permanently reports "1 service crashed" — alert fatigue on
the environment you most need to trust. Committing that staged deletion is one click
and one human.

---

## Stale ride root cause

Established by reading, not inferring. Four things combined:

1. **The rider cannot cancel a started ride.** Correct policy, and not the bug.
2. **`auto_cancel_trip` only covers trips nobody ACCEPTED.** It is scheduled once at
   creation with a countdown, so an accepted-then-abandoned trip has no timeout at
   all.
3. **`driver:active_trip:<id>` has no TTL**, and `sweep_stale_driver_presence` only
   removes geo entries — it never touches that key and leaves no durable trace.
4. **`add_driver_location` removes a driver from the geo index whenever that key is
   set.** So even after the driver reconnected and pinged, they were removed again
   every time. `drivers_notified=0`, permanently.

And underneath all of it: **the only liveness evidence that existed was
`driver:heartbeat:<id>` with a 45-second TTL.** Once it expired there was nothing
anywhere to say *when* activity stopped, so nothing could distinguish a driver who
dropped out a minute ago from one gone three hours.

## Stale ride recovery architecture

Deliberately **not** `TIMEOUT -> CANCEL`. Absence of connectivity is not evidence of
abandonment: a legitimate ride can cross a tunnel, sit in a basement car park, run
for hours in traffic, be backgrounded by an OEM battery manager, or end with a flat
battery while the driver is still carrying the passenger. Cancelling those would be
worse than the bug, because it would abandon real rides mid-journey.

```
DETECTION  ->  RECOVERY  ->  ESCALATION  ->  OPERATOR RESOLUTION
```

**Durable evidence** — four operational columns on `Trip`
(`last_driver_activity_at`, `last_rider_activity_at`, `stale_flagged_at`,
`stale_reason`), migration `0016`, additive. `last_driver_activity_at` is written
from the GPS path through a **conditional UPDATE coalesced to one write per
interval**: 12 pings a minute cost one UPDATE, not twelve, which at 50 concurrent
rides is the difference between 50 and 600 write transactions a minute. A write
failure returns False and never breaks the GPS frame handler.

**Classification** — `HEALTHY` / `STALE` / `RECOVERY_REQUIRED` /
`OPERATOR_ATTENTION`, pure and side-effect free, falling back through the lifecycle
timestamps so a trip predating the feature still classifies sensibly.
Thresholds configurable: `TRIP_STALE_AFTER_SECONDS` (600),
`TRIP_OPERATOR_ATTENTION_AFTER_SECONDS` (1800). Generous on purpose — they decide
when a human *looks*, never when the system acts.

**Recovery, automatic** — `reconcile_driver_active_trip` makes Redis agree with
PostgreSQL and runs on the driver's socket connect, before presence is announced.
It clears a stale marker when the database says the trip is over, reconstructs a
lost one when the database says it is live, and corrects one pointing at the wrong
trip. **PostgreSQL always wins.** This is a cache repair rather than a decision,
which is precisely why it is safe to automate — and it is what lets a crashed driver
return to supply with nobody involved. Recording activity also clears any stale flag,
so a ride that recovers leaves the queue by itself.

**Detection** — `ride.flag_stale_active_trips`, on beat every two minutes, flags and
does nothing else. Idempotent: an already-flagged trip is skipped, and the flag is
written under `select_for_update` with the status re-checked inside the lock, so a
Celery redelivery converges rather than re-alerting.

## Driver restart recovery

Proven by the drill's branch A (see **Abandoned ride drill**). Sockets are dropped
with code 1006 and no close frame — the way a killed app disappears, not the way a
polite client leaves — then the app restarts: fresh login, durable active-trip
query, reconnect, resume GPS, complete, confirm cash, and a probe booking to prove
the driver is back in supply.

`GET /ride/active/` was already the durable recovery endpoint and needed no change.
What was missing was the ephemeral repair, which now happens on connect.

**Phases 10 and 11 were not executed as separate timed drills** (30 s / 2 min / 5 min
outages, and permanent device loss). Branch A covers the crash-and-return shape;
the graded outage durations and the never-returns case are covered by branch B and
by the liveness tests rather than by timed QA runs.

## Rider restart recovery

**Phase 12 not executed.** `last_rider_activity_at` exists and is explicitly
advisory — a rider whose app is closed must never block a driver from completing a
legitimate ride — but no QA drill exercised rider-side restart this run.

## Operator stale ride workflow

New page in the canonical Django console at `/stale-rides/`, showing per row: trip,
status, ride length, silence, last driver activity, last rider activity, driver, a
**stable non-identifying rider reference**, the flag reason, and the assessment.
No phone numbers and no coordinates — an operator who must call someone opens the
ride; a queue is not a directory.

Two actions, both requiring a typed reason, both confirmed, both audited:

| Action | Effect |
|---|---|
| `mark_reviewed` | Records that a human judged the ride legitimate. Clears the flag. Changes nothing about the trip. |
| `release_driver` | Repairs the ephemeral availability marker **only when PostgreSQL already shows the ride finished**, so it cannot free a driver who is genuinely carrying a passenger. |

**Deliberately absent: `admin_complete` and `admin_cancel`.** Completing a ride
creates settlement, commission, a wallet movement and a receipt; cancelling a started
one raises what the rider owes. Neither policy has been decided for abandoned rides,
and **a control that invents money policy is worse than no control**. The page says
so in plain language rather than offering a dead end, and four tests assert those
actions are not accepted. Termination financial policy is **BLOCKED pending a
business decision** — exactly as the brief allowed.

The page also states, to the operator, that a flag is not a verdict and how to tell a
normal long ride from a temporary disconnect from a likely abandonment. Without that,
a queue of flagged rides reads as a list of broken ones and the safe-feeling default
becomes "cancel them all".

## Supply protection test

Phase 19, 8 tests, ten drivers:

| Claim | Result |
|---|---|
| Nine of ten remain dispatchable when one is stranded | **PASS** |
| The stranded driver is operationally visible, not silently missing | **PASS** |
| A healthy two-hour ride never appears in the queue | **PASS** (the control) |
| Several stranded rides are each flagged exactly once | **PASS** |
| Resolving the trip returns the driver to supply | **PASS** |
| The full fleet is dispatchable again afterwards | **PASS** |
| Reconciling one driver leaves the other nine untouched | **PASS** |
| A flagged trip still counts as active, so the driver gets no second ride | **PASS** |

The last one matters: restoring a stranded driver must be a consequence of resolving
their trip, never a workaround that hands them another one.

## Fare rounding

Root cause: each component was rounded independently while the total was rounded from
the unrounded subtotal, so `round(a)+round(b)+round(c) != round(a+b+c)`. QA trip 45
charged **135.22** while its lines read **60.00 + 55.59 + 19.62 = 135.21**.

`total_fare` is authoritative and **has not moved**. A rider is charged exactly what
they were charged before. What changed is the decomposition beside it.

Surge and the minimum fare break the sum legitimately — surge multiplies the
subtotal, the minimum fare replaces it — so presenting three metered lines as though
they explained such a total would be a different lie. Those uplifts became their own
named lines, and only what remains is called rounding, which is then at most one
paisa. All Decimal; no float touches money.

23 backend tests hold `base + distance + time + sum(adjustments) == total_fare`
exactly, across ten distances including thirds and paisa boundaries, four vehicle
types, and the minimum-fare shape. Plus: rounding is never more than a paisa, no
decorative zero lines, every adjustment carries a human label, and nothing
introduces `final_fare`.

Two of those tests initially pinned QA's rate-card figures and failed locally where
the defaults apply. **A unit test that depends on one environment's pricing is
testing the environment**, so they were replaced with an environment-independent
proof of the same property: the total must still equal aggregate-then-round rather
than the sum of the rounded components — which is precisely what would have changed
if the "fix" had quietly moved the price.

The rider panel renders the new lines and does no arithmetic of its own. The older
valueless "Minimum fare applied" flag stands down when the server sends an explicit
`minimum_fare` line, with a negative control proving the disclosure is not lost
against an older server. 5 widget tests including malformed payloads.

**Phase 21 (receipt arithmetic) was not executed.** The receipt renders its own
totals and GST; whether its components close exactly is untested and separate from
this.

## QA operator

Phase 22, resolved as option A. `QA_ADMIN_BOOTSTRAP_PHONE`, `_CODE` and `_PASSWORD`
had existed in QA for months with **nothing in the codebase reading any of them** —
which is worse than having none, because an entire previous run could not rehearse a
single operator workflow and the reason was that the capability was imaginary rather
than missing.

`manage.py bootstrap_qa_operator` now makes them real, with 16 tests holding the
properties that make it safe:

- **Refuses in production.** Not warns. Anything that cannot prove it is not
  production — an unlabelled `DEBUG=False` deployment — is treated as production.
- **No default password, no default phone.** A built-in admin password is how a test
  account becomes a production incident. Under 12 characters is refused.
- **Never emits the credential** — not to stdout, stderr, or the audit row. The audit
  row does not even carry the phone number.
- **Idempotent, and will not rotate a password nobody asked it to rotate.** A
  redeploy must not silently change a credential in use; rotation needs
  `--reset-password`.
- An account that drifted out of the operator role is repaired, not duplicated.
- The test that matters most proves the created account **can actually sign in and
  reach an operator-only page**.

`QA_ADMIN_BOOTSTRAP_CODE` is accepted and ignored so the variable set needs no
editing, and a test asserts nothing treats it as a second factor — because it is not
one.

**Phase 23 is one human action away:** run it once against QA. The variables are
already set there; their values are not known to me and are not needed to ship the
command.

## Ops workflow

**Phase 24 not executed.** The full operator drill — driver lookup, KYC queue,
document review, approve, reject, ride lookup, SOS, support, fare, settlement,
payout — still requires the account above to exist in QA. The stale-ride workflow
*was* exercised end to end, in tests, including that an unauthenticated caller
cannot reach the page.

## Ops security

**Phases 25, 26, 27 not executed.** No MFA, no rate limiting, no RBAC separation.
Unchanged and still launch-blocking: the console that approves KYC and releases
payouts is protected by a phone number and a password at the backend root.

**Phase 28 partially done.** Audit rows are written for the two new stale-ride
actions and for the operator bootstrap, with actor, action, target, timestamp and
reason, and tests assert the credential and phone number are absent. The pre-existing
KYC, payout, SOS and pricing audit coverage was not re-verified this run.

## Driver FCM

**Phases 30 and 31 not executed.** Unchanged and launch-blocking: the driver app
depends on `firebase_messaging`, the `google-services` plugin is commented out
because `google-services.json` is absent, `Firebase.initializeApp()` throws,
`PushService` catches it and sets `_ready = false`, and no token is registered. A
backgrounded driver receives no ride offer.

## Celery durability

**Phases 32 and 33 not executed as a kill-the-worker matrix.** Standing position
unchanged: `task_acks_late` and `task_reject_on_worker_lost` on globally, prefetch 1,
soft and hard time limits.

The **inventory guard earned its place again this run**: adding
`ride.flag_stale_active_trips` failed the suite until the task was registered with
an explicit durability justification. That is Phase 16's requirement working as
intended — it is now genuinely difficult to add a production task without stating
why late acknowledgement is safe for it.

**Phase 34: beat is still embedded in the worker** (`-B`), confirmed live in QA this
run. Safe only while exactly one worker runs, and it must be split before scaling.

## Redis failure/recovery

**Phase 35 not executed as a full matrix.** What exists: driver exclusivity under
genuine Redis unavailability (5/5, previous run), trip-creation coherence under
broker loss, SOS durability under broker loss, and now the reconciliation path.

**Phase 36 is materially advanced by this run.** Rebuilding ephemeral state from
PostgreSQL no longer needs a developer command: `reconcile_driver_active_trip` runs
on every driver socket connect, and an operator can trigger it for a specific driver
from the stale-ride queue. That was previously the exact thing that required a shell.

## SOS

**Phase 37 not executed** (it depends on the QA operator). Standing evidence: durable
through a broker outage, deduplicated, operator-visible via an endpoint that reads
PostgreSQL directly, coordinate-free logs. Proven live on the previous acceptance
ride and again on this run's final ride.

## Support

**Phase 38 not executed.**

## Receipts

**Phase 39 not executed.** Receipt resend returns 200 and economics remain unchanged
after retry, but S3 failure, email failure and duplicate-task behaviour are untested.

## GPS trust

**Phase 40 not executed.** `recorded_at` (device clock, documented untrusted) and
`received_at` (ours) are already kept separately and precisely because they
disagree, which is the right architecture — but the malicious-timestamp tests
(future, far past, out of order, duplicate, impossible speed) were not written.

## GPS retention

**Phase 41 not executed beyond the existing model.** ≈250–400 bytes per point
including index overhead, ≈200–300 points per urban ride: 2–4 GB/year at 100
rides/day, 180–440 GB/year at 10,000.

Worth restating: `GPS_TRAIL_ENABLED` is **False**, so zero durable points are stored
and `actual_distance_km` / `actual_duration_min` stay NULL. Confirmed again on this
run's rides. There is currently **no durable route evidence** for a dispute, an SOS
investigation or an insurance claim. That is a pilot decision nobody has made.

**Phase 42** is already covered by 10 pre-existing access-control tests including
cross-rider and cross-driver IDOR; not re-run.

## Load evidence

**Phases 43, 44, 45 not executed. There is still no concurrency evidence of any
kind.** This remains the largest untouched area, and nothing in this report supports
any claim about 10, 25 or 50 simultaneous rides.

## Database performance

**Phase 46 not executed.** No `EXPLAIN` plans captured. One index was added on
evidence of the query the detector runs (`last_driver_activity_at`,
`stale_flagged_at` are both `db_index=True`), which is a design decision rather than
a measurement.

## Mobile builds

**Phases 48, 49, 50 not re-run this session.** Standing state: both APKs build green
in CI with downloadable artifacts (rider ~95 MB, driver ~89 MB), rider 73 tests
passing / 7 skipped with analyze clean, driver 83 passing with analyze clean, ops
console lint + typecheck + build green.

**Phases 51, 52, 53 not executed** (branding sweep of generated metadata, production
env safety, merged-manifest permission review).

## Security

**Phase 54 not executed as a fresh sweep.** The pattern this project keeps producing
— infrastructure failure converted into a confident business answer — has now
produced six instances across runs (Redis availability, payout double-payment, OTP
false success, booking false failure, the inner payout handler, and the stale
active-trip marker). The remaining ~50 classified-but-unchanged broad handlers were
not revisited.

## Secrets

**Phase 55 not re-verified.** Both credentials remain **unrotated**: the
`POSTGRES_PASSWORD` committed in `9af854c` on 2026-05-07 (~4.5 months, in every
clone), and the Google Maps key shared between `AndroidManifest.xml` and
`AppDelegate.swift` and therefore un-restrictable on either platform. Production
stays RED on secrets.

## Dependencies

**Phase 56 not re-run.** 14 advisories across `urllib3 2.0.7` and
`sentry-sdk 1.32.0`, all blocked by `cashfree-pg==3.2.12` declaring
`urllib3 <2.1.0` and `sentry-sdk <1.33.0`. Independently fixable: zero.

## Cashfree

**Phase 57 not executed.** The sandbox test plan from the previous report stands,
with `transferId` idempotency still the linchpin for every retry story.

## Production preparation

**Phases 58, 59, 60 not executed this run.** Production still cannot boot, still has
zero application variables, and **still tracks `dev`**. The boot-guard rehearsal
(15 subprocess tests) and the environment matrix remain current.

## Backups

**Phase 61 not executed. Production remains RED on recovery.**

## Rollback

**Phase 64 not executed.**

## Final acceptance ride

**30 of 30 stages PASS** on trip 46 against revision `34328ef`, deployments frozen.
Full evidence in the appendix.

## Abandoned ride drill

**Branch A: 9 of 9 stages PASS** on trip 47 -- a driver crashes mid-ride, recovers
by itself, completes the ride and returns to supply with nobody involved. Branch B
was not run as a timed QA drill; its harness is committed and the behaviour is
covered by 51 tests. Full evidence in the appendix.

---

## Genuine launch blockers

Reclassified per Phase 70: RED only where it can break a core ride, lose or corrupt
money, strand supply without recovery, break safety, expose serious security, prevent
operators from operating, or prevent deployment.

1. **Production cannot boot and has no configuration.** Prevents deployment.
2. **Production tracks `dev`.** Repoint before configuring, or the first push after
   configuration ships whatever a developer committed.
3. **Two credentials compromised and unrotated** — database password (~4.5 months)
   and the shared Maps key.
4. **Ops console has no MFA and no rate limiting**, in front of KYC approval and
   payout release.
5. **The driver app registers no FCM token** — a backgrounded driver gets no offer.
6. **No QA operator account exists yet** (one command away), so no operator workflow
   beyond stale rides has been rehearsed.
7. **An `unresolved` payout still cannot be resolved** by anyone.
8. **No load evidence of any kind.**
9. **Backups unverified.**
10. **14 dependency advisories**, all behind the Cashfree pin.

**No longer a blocker:** a stranded in-progress ride. It is detected, recoverable by
the driver without help, visible to operations, and resolvable by an operator —
proven by 51 tests and the drill.

## Technical debt

- Beat embedded in the worker (`-B`).
- `alluring-happiness`: crashed stray QA service, staged-delete never committed,
  source of a permanent false alarm.
- Termination financial policy for abandoned rides undecided, so the operator has
  detection and release but not termination.
- Receipt component arithmetic unverified (Phase 21).
- `GPS_TRAIL_ENABLED=False` — no durable route evidence; a decision, not a bug.
- Driver "available balance" shows negative for cash-heavy drivers with no
  explanation.
- One postgres test still needs its own process.
- ~50 classified-but-unchanged broad exception handlers.

## Human actions

1. **Rotate the leaked PostgreSQL password**, then update the consuming
   configuration. Rotation ends the exposure; removing it from HEAD does not.
2. **Repoint the production Railway service off `dev`** to a tag or manual trigger —
   *before* populating any variable.
3. **Run the QA operator bootstrap once:**
   `railway run --service backend --environment QA python manage.py bootstrap_qa_operator`
   This unblocks every remaining operator rehearsal.
4. **Put the console behind MFA** (identity proxy in front of Django is the least
   invasive route), and **rotate and split the Maps key** per platform.
5. **Create `google-services.json` for both apps** and re-enable the
   `com.google.gms.google-services` plugin in the driver app, then commit the staged
   deletion of `alluring-happiness` while in the Railway console.

---

## Launch decision matrix

RED only where it can break a core ride, lose or corrupt money, strand supply
without recovery, break safety, expose serious security, prevent operators from
operating, or prevent deployment. Technical debt is kept separate.

| Item | Status | Evidence |
|---|---|---|
| Stranded in-progress ride | **GREEN** | Detected, self-recovered by the driver, operator-visible and operator-resolvable; 51 tests plus the drill. Was this run's Priority 1. |
| Driver crash recovery | **GREEN** | Drill branch A: sockets dropped with code 1006, app restarts, trip recovered from `/ride/active/`, ride completed, driver back in supply. |
| Supply protection | **GREEN** | Nine of ten drivers stay dispatchable with one stranded; full fleet restored on resolution. |
| Operator stale-ride workflow | **GREEN** | `/stale-rides/` with two audited, reason-required actions; 19 tests including that an unauthenticated caller cannot reach it. |
| Abandoned-ride termination policy | **BLOCKED** | Money policy undecided; no admin-complete or admin-cancel exists, and four tests assert they are not accepted. |
| Fare decomposition | **GREEN** | base + distance + time + adjustments == total exactly, 23 tests; the total provably unmoved. |
| Rider fare panel | **GREEN** | Renders the server's adjustment lines and does no arithmetic; 5 tests including malformed payloads. |
| Ride lifecycle | **GREEN** | Five correlated committed acks on the final ride. |
| Cash economics | **GREEN** | Reconciles exactly; one earnings row; retries return already_done. |
| Driver exclusivity | **GREEN** | 5/5 with Redis genuinely dead, second transaction provably blocked. Unaffected by this run. |
| SOS | **GREEN** | Durable, deduplicated, operator-visible, coordinate-free; proven live again. |
| Redis reconciliation | **GREEN** | PostgreSQL wins, automatic on connect, operator-triggerable; no shell required. |
| Celery inventory guard | **GREEN** | Blocked this run's new task until its durability was justified. |
| QA operator bootstrap | **YELLOW** | Implemented with 16 tests including a real sign-in; one human command from existing. |
| Celery worker-loss matrix | **YELLOW** | Configuration correct and guarded; no kill-the-worker test run. |
| Beat topology | **YELLOW** | Still embedded via -B; safe at one worker only. |
| GPS durable trail | **YELLOW** | Live relay proven; trail disabled, so no route evidence and no actual metrics. |
| Receipt | **YELLOW** | Resend clean and economics stable; failure paths and component arithmetic untested. |
| Rider APK | **GREEN** | CI green on 719736a. |
| Driver APK | **GREEN** | CI green on bae0a3e, 89 MB artifact. |
| Ops console build | **GREEN** | lint, typecheck and build green. |
| Ops MFA | **RED** | Not implemented. |
| Ops rate limiting | **RED** | Not implemented. |
| Ops RBAC | **RED** | Support and payout authority not separated. |
| Ops audit | **YELLOW** | New actions audited with actor, target and reason; pre-existing KYC, payout, SOS and pricing audit not re-verified. |
| Driver background offer | **RED** | No google-services.json; no FCM token registered. |
| Payout ambiguity | **GREEN** | An unknown outcome holds the money; 11 tests with both negative controls. |
| Payout resolution | **RED** | Unresolved can be entered and not left. |
| Online payment | **BLOCKED** | Cashfree three majors behind; provider contract unverified. |
| Dependencies | **BLOCKED** | 14 advisories, zero independently fixable. |
| Secrets | **RED** | Two compromised credentials unrotated; nothing sealed. |
| Load | **RED** | No measurement of any kind. |
| PostgreSQL performance | **YELLOW** | No EXPLAIN plans; the one new index is a design choice, not a measurement. |
| Backups | **RED** | Unknown whether any backup exists. |
| Rollback | **RED** | Never rehearsed. |
| Production config | **RED** | 8 variables, all Railway-injected. |
| Production deployment | **BLOCKED** | Cannot boot; tracks dev. |
| Observability | **YELLOW** | Structured events now include stale_active_trip_flagged and driver_active_trip_repaired; no dashboards or alerts. |
| App distribution | **RED** | No release signing; two package IDs at different organisations. |
| India legal/business | **RED** | Receipt legal entity and grievance mailboxes unconfirmed. |

---

## Final questions

### Can a driver recover from an app crash during a ride without engineering?

**YES.**

Drill branch A: the driver's sockets are dropped with code 1006 and no close frame,
the app restarts with a fresh login, `GET /ride/active/` returns the trip,
reconciliation clears the stale availability marker on socket connect, GPS resumes,
and the ride completes and settles normally. A probe booking afterwards confirms the
driver is dispatchable again. No operator and no engineer are involved at any point.

### Can operations recover a genuinely abandoned in-progress ride without engineering?

**YES, within the bounds that have been decided.**

An operator sees the ride at `/stale-rides/` with the evidence to judge it, and can
mark it reviewed or release the driver's availability. Both are audited, both require
a reason, and the release is refused when the ride is genuinely still active.

What they cannot do is terminate the ride, because completing or cancelling an
abandoned ride is a money-policy decision nobody has made. That is BLOCKED
deliberately, not missing by oversight. So: supply is recoverable without
engineering; final disposition of the trip record is not yet.

### Can one stranded driver poison dispatch supply?

**NO.**

Eight tests with a ten-driver fleet: nine remain dispatchable, the stranded one is
visible rather than silently absent, a healthy long ride is never flagged, and the
full fleet returns on resolution. Reconciling one driver leaves the other nine
untouched.

### Can we onboard the first 10 drivers through normal operations?

**NOT YET.**

Minimum remaining condition: run the QA operator bootstrap (one command), then MFA on
the console, then google-services.json for the driver app so a driver actually
receives offers, then one APK installed and driven on a real handset.

### Can we safely execute 100 controlled cash rides?

**NOT YET.**

The ride works end to end and the money reconciles exactly, and the stranded-ride
failure that would certainly have occurred across 100 rides is now handled. What
remains is somewhere to run them, because production cannot boot, and no load
evidence at any concurrency.

### Can we enable online payments?

**NOT YET.**

Minimum remaining condition: the Cashfree sandbox suite passing, starting with
transferId idempotency and webhook signature verification against 6.x.

### Can we automate real payouts?

**NOT YET**, and it should stay that way for the pilot.

Ambiguity is handled correctly: an unknown provider outcome holds the money rather
than refunding it. But unresolved has no exit, and automatic resolution needs a
provider status call that has never been verified. Minimum remaining condition: the
operator resolution flow, then the sandbox proof.

### Can we deploy production?

**NOT YET.**

Minimum remaining condition, in order: rotate the leaked database password; repoint
the production service off dev; populate the production variables sealed; confirm a
backup exists and restore it once. The boot guards will then name anything still
missing.

---

## Next actions

### Engineering actions — next 5

1. **Load harness at 10, 25 and 50 concurrent rides** with realistic GPS against
   isolated infrastructure. This is now the largest area with no evidence at all.
2. **The unresolved-payout resolution flow** in the console: show the transferId,
   require a human to record what Cashfree reports, then move the row to processed or
   failed, refunding only on a confirmed failure.
3. **Driver FCM registration** — permission, token, backend registration, refresh,
   logout, invalid token — plus the background-offer path. Implementation and tests
   are achievable now; the push proof needs real credentials.
4. **Celery worker-loss matrix and Beat separation**: kill a worker mid-task for
   auto_cancel_trip, receipt, GPS persistence and actual metrics, prove redelivery
   and duplicate safety, then split beat out of the worker.
5. **Ops rate limiting and RBAC**, so a support agent does not inherit payout
   authority and the login cannot be brute-forced.

### Human actions — next 5

1. **Rotate the leaked PostgreSQL password.** Everything else can wait; this cannot.
2. **Repoint the production Railway service off `dev`** — before populating any
   variable.
3. **Run `bootstrap_qa_operator` against QA once.** It unblocks every remaining
   operator rehearsal, and the variables it needs are already set there.
4. **Put the ops console behind MFA**, and rotate and split the Maps key per
   platform.
5. **Create `google-services.json` for both apps**, re-enable the driver's
   google-services plugin, and commit the staged deletion of `alluring-happiness`.

---

# Appendix — final acceptance ride and abandoned-ride drill

Run with deployments frozen against QA revision `34328ef`, verified through
`/version` (`revision_source: RAILWAY_GIT_COMMIT_SHA`) rather than a 200 from
`/healthz`.

## Final acceptance ride (Phase 68)

**30 of 30 stages PASS.** Trip **46**, real QA PostgreSQL, Redis, Daphne/Channels and
Celery, no mocks on the main path.

| Command | `command_id` | ack | resulting status |
|---|---|---|---|
| accept | `accept-9eca28296e` | **committed** | accepted |
| reached | `reached-f20c45afc7` | **committed** | reached |
| start | `start-1530c92299` | **committed** | in_progress |
| complete | `complete-236eaacf13` | **committed** | completed |
| confirm_cash | `confirm_cash-45d6cbbe17` | **committed** | completed |

```
quote                    169.02
ride duration            154 s
GPS frames sent          30
relayed live to rider    30  (30/30)
SOS                      sos_id=2, status=open, HTTP 201
SOS second press         HTTP 200, repeat_of=2   (deduplicated)
fare snapshot            base_fare, distance_fare, time_fare,
                         surge_multiplier, total_fare
final_fare               None
rider history            trip 46 present
```

Retries (Phase 4 equivalence):

```
retry complete            ack=already_done    status=completed
retry confirm_cash        ack=committed       status=completed
retry receipt             HTTP 200
estimated_fare unchanged  True
final_fare still null     True
duplicate wallet credit   none
```

The one stage that failed on the previous ride — `FARE_SNAPSHOT` — now passes. That
was a harness fault (it read `/trip/<id>/details/`, the driver-details endpoint,
instead of `/trip/<id>/`), fixed and committed.

## Final money proof (Phase 68)

Computed from the immutable trip evidence, not from today's rate card:

```
gross (charged)          169.02
commission                33.80    = 20.00 % of gross
driver net               135.22
gross - commission       135.22    -> matches net exactly
final_fare               None
earnings rows for 46     1
trips with >1 row        none, across the whole feed
```

## Abandoned-ride drill, branch A (Phase 69) — the key new launch proof

**9 of 9 stages PASS.** Trip **47**, driver sockets dropped with code 1006 and no
close frame — the way a killed app disappears, not the way a polite client leaves.

```
SETUP_RIDE_IN_PROGRESS        trip 47, ack=committed, status=in_progress
DRIVER_CRASHED                sockets dropped without a close

A_TRIP_SURVIVES_THE_CRASH     status still in_progress   <- no false auto-cancel
A_DRIVER_REAUTHENTICATES      fresh login
A_DRIVER_RECOVERS_ITS_TRIP    /ride/active/ returned trip 47  (expected 47)
A_DRIVER_RESUMES_GPS          reconnected and pinging
A_RIDE_COMPLETES_NORMALLY     ack=committed, status=completed
A_CASH_CONFIRMED              ack=committed
A_DRIVER_BACK_IN_SUPPLY       drivers_notified=1, probe trip 48
```

The last line is the one that matters. `drivers_notified=0` there would have been
trip 42 reproducing exactly. **No operator and no engineer were involved at any
point in that sequence.**

### Branch B

Not executed as a timed QA run. It needs silence beyond the deployment's
`TRIP_STALE_AFTER_SECONDS` (600) for the flag to appear, and the harness for it is
committed (`qa/abandoned_ride_drill.py --branch b --wait 700`). The behaviour it
would assert — no false auto-cancel, `final_fare` still NULL, PostgreSQL still
holding the truth, and the trip becoming visible as stale — is covered by 51 unit and
integration tests including the explicit negative controls that the detector changes
no status and touches no money.

## A defect found by this verification, and fixed

Checking the live quote for the new fare adjustment lines found them **absent from
the API entirely**: base + distance + time came to 135.21 beside a charged 169.02,
with the surge uplift unexplained — the exact defect the fare work was meant to
close.

There are **three** places that rebuild that payload, not two. `quote_fare` produced
the lines, `estimate_amount` was taught to forward them, and the estimate-fare
**view constructs its own response dict field by field** and dropped them. Twenty-three
unit tests against the service and the wrapper structurally could not see the third
layer.

Fixed, with 11 tests that go through the HTTP API instead: the field is present, the
decomposition closes exactly across five distances, each amount is a JSON-safe string
(a float would put money into binary floating point on the way to the rider), labels
leak no internal names, and the quoted total is unchanged.

Same shape as the harness faults from the proof run — a check that passed while the
thing it verified did not work — and the same lesson: **verify at the boundary a
consumer reads, not the layer that is convenient to call.**

## Core success condition

Demonstrated on QA at revision `34328ef`, with no engineering intervention:

```
DRIVER CRASHES MID-RIDE
  |
  +-- driver reconnects  ->  trip restored (47)  ->  ride continues  ->  completed
  |                                                   cash confirmed
  |                                                   driver back in supply (probe 48)
  |
  +-- driver does not return  ->  detector flags (tested)
                               ->  ops sees it at /stale-rides/ (tested)
                               ->  reviewed action, or driver released (tested, audited)
                               ->  driver supply restored (tested)

while:
  PostgreSQL remained the truth            (reconciliation always defers to it)
  Redis was reconstructable                (repaired on connect, no shell)
  no money was invented                    169.02 - 33.80 = 135.22, one earnings row
  no trip was falsely completed            trip 47 stayed in_progress until the
                                           driver completed it
  no rider was charged by a detector       the detector changes no status and
                                           touches no money, asserted by test
  no developer shell was required          at any point in branch A
```

The right-hand branch is proven by tests rather than by a timed QA run, and the
report says so rather than implying otherwise.

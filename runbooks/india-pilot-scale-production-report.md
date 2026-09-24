# India pilot — scale and production readiness report

**Run scope:** OPERATE → SECURE → NOTIFY → LOAD → FAIL → RECOVER → PREPARE PRODUCTION
**Backend revision at end of run:** `8181e2c` on `dev`
**QA deployment verified through:** `GET /version` (`revision_source: RAILWAY_GIT_COMMIT_SHA`)
**Gates at end of run:** 901 passed, 9 skipped, 102 deselected; `ruff` clean

---

## 0. Read this first

Three things found this run change the launch conversation. They are stated here
rather than in their workstream sections, because a reader who stops after one page
should still have them.

**1. Anyone able to receive an SMS could make themselves a platform operator.**
Demonstrated end to end, not inferred. One OTP request carrying `"role": "admin"`, one
login, and the resulting account read the driver KYC queue, the payout queue, every
trip, live driver positions and the operations dashboard — all HTTP 200. Two
independent defects had to line up, and both are now closed with 41 tests, 11 of which
fail against the pre-fix code.

It was open on QA for most of this run. **Verified closed on the running QA deployment**
at revision `8181e2c`:

```
POST /api/v1/auth/otp/ {"role": "admin"}  -> HTTP 400  AUTH_INVALID_ROLE
                                             "Role must be one of ['rider', 'driver']"
POST /api/v1/auth/otp/ {"role": "rider"}  -> HTTP 200  (control: signup still works)
```

It was never exploitable in production, for the reason described in point 3.

**2. `acks_late` was not buying what it appeared to buy.** With the Redis transport, a
task whose worker dies is not lost — but it is invisible to every other worker until
`visibility_timeout` elapses, and that was unset, so kombu's default of 3600 seconds
applied. `auto_cancel_trip` fires 90 seconds after a trip is created. On the default it
could have been an hour late. Now configured to 900 s from a derived floor; measured
recovery is the timeout plus up to ~100 s of restore-poll granularity.

**3. There is no production environment.** The Railway `production` environment
contains exactly one service. It is marked deleted, has a destructive uncommitted
staged change against it dating from 2026-09-22, holds zero environment variables, and
has no domain. There is no production PostgreSQL, no Redis, no Celery worker and no ops
console. Separately, that service is wired to build from the `dev` branch, so every
push during this run triggered a production build. All of them failed, and the reason
they failed is itself a finding — see §20.

---

## 1. What this run did and did not do

Six workstreams were opened. All six received evidence. None was completed, and the
sections below say which parts are proven, which are tested-but-not-rehearsed, and
which are blocked on a human.

| | Workstream | Evidence produced | Blocked remainder |
|---|---|---|---|
| A | Operations | Branch B stale-ride drill 5/5 on QA | Operator login (§2) blocks A1, A2, A4–A8 |
| B | Security + notifications | Privilege escalation closed; push registration fixed | MFA, RBAC granularity, Maps key |
| C | Celery + Redis | Kill-the-worker matrix 14/14 on Linux; 12 duplicate-safety tests | Redis outage matrix, beat topology |
| D | Load + performance | Query plans at 20k trips; pagination at HTTP boundary | Concurrent-ride load, percentiles |
| E | Production + release | Full production audit; CVE audit | Everything else — production does not exist |
| F | Final pilot proof | See §32 | Depends on A and E |

---

## 2. RULE 1 — the QA operator bootstrap could not be run

The instruction was to run the existing QA-only `bootstrap_qa_operator` against QA as
the first command. It could not be run, by two independent paths:

- **Railway CLI is unauthenticated.** `whoami` and `status` exit 1 with no output, so
  `railway run --service backend --environment QA python manage.py
  bootstrap_qa_operator` cannot execute. Authenticating needs a human at a browser or a
  device code; there is no unattended path.
- **The Railway MCP server is authenticated but returns variable names only.**
  `list-variables` for the QA backend service reports `valuesRedacted: true`
  (OAuth-connected apps receive names, not values). `QA_ADMIN_BOOTSTRAP_PHONE`,
  `QA_ADMIN_BOOTSTRAP_CODE` and `QA_ADMIN_BOOTSTRAP_PASSWORD` are confirmed **present**
  in QA. Their values are not readable, and no MCP tool runs a command in a service.

So the command is shipped, tested (16 tests) and ready, and it has still never been
run. The consequence is not cosmetic: **every operator workflow rehearsal in workstream
A remains unperformed**, because there is no account to sign in with and the password
is not knowable from here. That is one human action — see §30, Human item 1.

Per the instruction, non-operator work continued immediately rather than waiting.

---

## 3. Workstream A — operations

### A3. Stale-ride Branch B, as an actual timed drill (done)

The half that had only ever been proven by unit tests is now proven against QA.

```
revision under test  d83fb19, environment qa
trip                 49
SETUP_RIDE_IN_PROGRESS      ack=committed, status=in_progress
SETUP_GPS_SENT              6 frames
DRIVER_GONE                 sockets dropped, no close frame
WAITING_FOR_DETECTION       720 s of silence

B_NO_FALSE_AUTO_CANCEL      PASS   status still in_progress
B_MONEY_UNTOUCHED           PASS   final_fare = None
B_POSTGRES_HOLDS_THE_TRUTH  PASS   trip 49

DRILL STAGES PASSED 5/5
```

Twelve minutes of total silence on an `in_progress` ride, and nothing cancelled it,
nothing charged anybody, and PostgreSQL still held the truth. That is the property the
trip-42 work existed to establish, and it is now observed rather than argued.

**What this drill still does not prove:** that the trip became *visible as stale in the
operator queue*. Staleness is exposed only through `servers/admin_dashboard` views,
which require an operator session — so verifying it needs §2 resolved. The detection
half remains covered by 51 unit and integration tests, including the negative controls
that the detector changes no status and touches no money. The report says so rather
than implying a full end-to-end.

### A1, A2, A4–A8 (blocked)

Operator login rehearsal, representative driver onboarding, the SOS operator path,
support tooling with a cross-user IDOR test, financial investigation without SQL,
unresolved-payout presentation, and usability fixes all require signing in to the
console. All blocked behind §2. Nothing was substituted or simulated, and none of them
should be marked anything but BLOCKED.

One A-workstream item *was* advanced without console access, because it is an
authorization question rather than a workflow one — see §5.

---

## 4. Workstream B — the privilege escalation (fixed)

### What was demonstrated, before any change

```
POST /api/v1/auth/otp/   {"phone_number": X, "role": "admin"}  -> 200
POST /api/v1/auth/login/ {"phone_number": X, "otp": ...}       -> 200
created user: role='admin' is_staff=False is_superuser=False

/api/v1/driver/admin/              -> 200   driver KYC queue
/api/v1/driver/admin/withdrawals/  -> 200   payout queue
/api/v1/ride/admin/trips/          -> 200   every trip
/api/v1/ride/admin/live-locations/ -> 200   live driver positions
/api/v1/ride/admin/dashboard/      -> 200   operations dashboard
```

A fresh phone number and a single OTP. The account could approve driver KYC, read the
payout queue, and see where every driver was.

### The two defects

**`role` was something a client asked for.** `'admin'` was in `VALID_ROLES`, and
`request_otp` validated the caller's requested role against that list. The role is
cached there and `/login/` mints the account with it.

**`IsAdmin` did not check `is_staff`** — although its own docstring listed it among
"three conditions, all required". The second gate existed in documentation and nowhere
else, and it is precisely the gate that would have stopped this, because `is_staff` is
not settable by any request.

Either defect alone would have prevented the escalation. Both are closed, because
fixing one would have left the system a single edit from the same hole.

### The fix

- `SELF_SERVICE_ROLES = ['rider', 'driver']`. `'admin'` remains a role the system has;
  it is no longer a role a client may ask for. The refusal is identical to the refusal
  for a nonsense role and does not name `admin`, so it is not a probe for which
  privileged roles exist. Re-checked again at the point of user creation, because
  `role` arrives from a different request's cache and that line is what mints the
  account.
- `IsAdmin` enforces `is_staff`. This is the right gate: it is already how the rest of
  the codebase identifies operators (`sos.dispatch_sos` fans out to
  `filter(is_staff=True, role='admin')`), and it is set only by
  `bootstrap_qa_operator` and the Django admin.
- The ops console had **three** separate copies of the lax expression. They are now one
  `is_operator()`, and a test asserts the console and the API return the same answer
  for all five combinations of role / `is_staff` / `is_superuser`.

### Evidence

41 tests, all calling the endpoints directly with real tokens. **11 fail against the
pre-fix code.** Three are controls and matter as much as the rest: a real operator
still reaches every endpoint, a superuser still does, and rider/driver signup still
works — without them, an `IsAdmin` that refused everyone would have passed everything
else and the "fix" would have been an outage.

---

## 5. B4 — authorization at the API, not in the layout

The instruction was that frontend hiding is not authorization. Every test in §4 holds a
token and calls the endpoint itself. Specifically asserted:

- a driver calling their own KYC-approval endpoint is refused, and is not approved;
- a rider calling payout approval is refused — and the assertion accepts only 401/403,
  because a 404 there would mean authorization is decided *after* the object lookup;
- an account with `role='admin'` and `is_staff=False` is refused payout approval;
- anonymous is refused every operator endpoint (this previously existed for the console
  only, via `test_admin_dashboard_authz.py`; it now exists for the API too).

---

## 6. B3 — RBAC does not exist yet (RED, and it is a design question)

The requirement was "support must not gain payout authority". The finding is prior to
that: **there is no support role.** `role` has exactly three choices — rider, driver,
admin — so every operator is a full admin who can approve KYC *and* release payouts
*and* read every trip and location.

This is not a gap in an RBAC implementation. There is no RBAC implementation. Building
one is in merge authority, but it is a design decision with a real product surface (who
sees rider phone numbers? who can reverse a KYC decision? is there a maker-checker on
payouts?) and shipping a half-wired version into a console nobody can currently sign
into would be worse than shipping none. It is listed as Engineering item 2 in §29 with
the shape it should take.

For a ten-driver pilot with one or two trusted operators, a single admin role is a
defensible interim position **provided** §2 is resolved so those accounts exist
deliberately, MFA is on, and the escalation path from §4 stays closed. That is a
judgement for the launch decision, not something this report can settle.

---

## 7. B5 — driver FCM registration (fixed)

Three defects in the four lines of the login view that register a device for push.

**A login without a device token erased the stored one.** `device_token` is optional
and defaults to `None`, and the assignment was unconditional. The driver app's own
`PushService` documents four ordinary states in which it has no token to send — no
Firebase config, no Play Services, permission not yet granted, no network at launch —
and it omits the field in each (`'device_token': ?deviceToken`, so the key disappears
rather than arriving null).

A driver who signed in during any of those states lost push, and stayed without it
until some later login happened to carry a working token. Push is the fallback dispatch
channel: it is what buzzes a driver whose socket is down with the screen off. Nothing
raised, nothing logged, nothing visible to an operator — just a driver who stops
getting background offers and cannot say why.

**A bare `save()` wrote every column.** On the login view's error path the user is
re-fetched after a failed create, so the instance can be stale, and a full-row write
from a stale instance is the same shape as the defect that used to revert trips drivers
had already accepted. Now `update_fields=['fcm_token']`, asserted by a test that
inspects the call.

**The token was echoed back in every user payload.** An FCM token is a device
credential — holding one is enough to push to that handset. It was in
`UserModelSerializer.fields` and not write-only, so it came back in the login response
and in any admin view of a driver's profile. Now write-only. All three clients (driver,
rider, ops web) were checked first: none reads the field, so no client contract
changed.

Registration is also no longer fatal — a failed write does not 500 the login, because a
driver who cannot sign in cannot work, whereas a driver signed in without push still
receives every offer in the foreground.

11 tests at the HTTP boundary through the real two-step OTP flow. **6 fail against the
pre-fix code.**

### Still open on B5

The driver app registers a token **at login only**. FCM rotates tokens (reinstall,
restore, Firebase-initiated refresh) and there is no `onTokenRefresh` listener, so a
driver who stays signed in for months can be holding a token the server no longer has.
That needs a mobile change plus a refresh endpoint. Engineering item 1 in §29.

---

## 8. B6 — background ride offer (YELLOW, and it must stay YELLOW)

The instruction was explicit: do not mark this GREEN from unit tests alone. It is not
marked GREEN. The server-side path is tested and the client-side handler exists and is
built to degrade quietly, but **no real device has been observed receiving a
backgrounded ride offer from a push**, so nothing here is proof that a driver with a
dark screen gets buzzed. That requires a physical handset, a real Firebase project and
someone watching it. Human item 3 in §30.

---

## 9. B2 — ops login brute force (done, previous run, unchanged)

Measured against QA before the fix: 12 consecutive failed logins in 7.8 seconds with no
throttling, on the console that approves KYC and releases payouts. Now keyed on both
phone and source IP, with an identical refusal message for a locked account, a wrong
password and an unknown phone so a lockout cannot be used to discover which numbers are
real. 10 tests at the HTTP boundary through the real form with a real CSRF token.

The guard **fails open** on a cache error, deliberately and with the tradeoff asserted
by a test: the cache is Redis, and locking every operator out during a Redis incident
is worse for a ten-driver pilot than temporarily losing brute-force protection. That is
recorded as a known, chosen weakness rather than presented as a complete defence.

---

## 10. B8 — credential hygiene recheck

No credential is printed by any harness added this run. Specific properties asserted by
test rather than by inspection:

- the FCM token never reaches the logs, on the success path or the failure path;
- the push token is absent from the login response body;
- `bootstrap_qa_operator` writes no phone number and no credential to its audit row;
- the ops login guard logs a locked IP without the phone numbers it was trying.

Unchanged from previous runs and still true: any credential that has ever appeared in
the repository must be treated as compromised and rotated, and removing it from HEAD is
not rotation.

---

## 11. B9 — Maps API key (RED, unchanged)

No work this run. Still requires a restricted, referrer- or app-bound key per platform
and a decision about billing exposure before either mobile app ships to a store.

---

## 12. Workstream C1 — the kill-the-worker matrix (done)

Previously deferred repeatedly. Executed against a real broker with real worker
subprocesses and real `SIGKILL`.

```
platform linux, kill = SIGKILL (worker exit -9)

BROKER_REACHABLE                      PASS
CONTROL_TASK_RUNS / RUNS_ONCE         PASS   undisturbed execution, exactly once
CONTROL_QUEUE_DRAINS                  PASS
TASK_REACHES_THE_WORKER               PASS
RESERVED_MESSAGE_IS_HELD_UNACKED      PASS   unacked hash holds 1 while executing
WORKER_DIES_MID_TASK                  PASS   killed before the finish marker
TASK_DID_NOT_COMPLETE                 PASS   work genuinely interrupted
MESSAGE_SURVIVES_THE_KILL             PASS   unacked still holds 1 after death
REPLACEMENT_WORKER_STARTS             PASS
TASK_IS_REDELIVERED                   PASS   105 s after the kill
REDELIVERY_IS_BOUNDED_BY_THE_TIMEOUT  PASS
REDELIVERED_TASK_COMPLETES            PASS

14/14 assertions, 5 observations
```

A purpose-built probe task holds the execution window open, because every real task in
this project finishes in milliseconds and there is no reliable moment to kill one. The
probe is labelled a probe, and its docstring states what it proves — the broker
contract — and what it does not: any product task's safety. Conflating those is how a
green suite ends up asserting less than it appears to.

### C1 finding 1 — `visibility_timeout` was unset

`acks_late` means "not lost". It does not mean "redelivered promptly". With the Redis
transport a reserved message is moved out of the queue into an `unacked` hash with a
deadline in `unacked_index`, and it is invisible to every other worker until that
deadline passes. Unset, kombu's default is **3600 seconds**.

`auto_cancel_trip` is scheduled 90 seconds after a trip is created. On the default, a
worker restart in that window meant the rider could wait an hour, still watching a
search that had already been given up on.

Now `CELERY_BROKER_TRANSPORT_OPTIONS = {'visibility_timeout': 900}`. The floor is
derived, not chosen:

```
longest countdown    180 s   TRIP_ACTUALS_DELAY_SECONDS — an ETA message sits
                             unacked in worker memory for the whole countdown
longest execution    360 s   CELERY_TASK_TIME_LIMIT
                   -------
worst legitimate     540 s   below this, recovery becomes a duplicate-execution
                             generator: a second worker takes a message the first
                             is still working on
```

Three tests pin the derivation, so raising `TRIP_ACTUALS_DELAY_SECONDS` or
`CELERY_TASK_TIME_LIMIT` cannot silently invalidate the timeout.

**The recovery SLA is therefore ~15–17 minutes, not 90 seconds.** Measured redelivery
was 105 s with the timeout set to 30 s; the extra 75 s is restore-poll granularity —
kombu's `restore_visible()` returns early unless `(count - 1) % 10 == 0` and the event
loop calls it every 10 s, so a real attempt happens roughly every 100 s. This is stated
rather than glossed because an operator needs to know that a worker crash means a
quarter-hour, not a minute.

### C1 finding 2 — a Celery worker on Windows never recovers anything

`WorkController.should_use_eventloop()` ends with `and not self.app.IS_WINDOWS`. A
Windows worker runs `synloop`, `register_with_event_loop` is never called, and that is
the only place the periodic restore is scheduled. The message sits in `unacked`
forever. Confirmed by calling `qos.restore_visible()` by hand against the same broker,
which restored it immediately.

Production runs Linux containers, so this is a **constraint on where a worker may
run**, not a production defect. The drill prints its platform in its header so a
Windows result can never be read as proof of redelivery — the first run of this drill
reported "not redelivered" and that result was a platform artifact, not a bug.

---

## 13. Workstream C2 — duplicate execution (done)

At-least-once delivery is the bill for not losing work. Twelve tests execute the six
ride/money/safety tasks twice through the task machinery and compare durable state.
"Safe" is not one behaviour, and the tests say which one each task has:

| Task | Behaviour | Asserted |
|---|---|---|
| `auto_cancel_trip` | converges | one cancellation, one rider notification, `cancelled_at` not rewritten; a redelivery arriving after an acceptance stands down without clearing `driver_id`; a completed trip is never cancelled |
| `compute_trip_actuals` | converges | same actuals; `final_fare` stays NULL however many times it runs |
| `issue_receipt_for_trip` | converges | one receipt, same number and version — plus the unique index that makes a *concurrent* duplicate impossible, since the guard itself is a read-then-create two workers can interleave |
| `persist_location_trail` | cannot duplicate | `(trip, source_event_id)` is refused by the database, which is what makes re-processing a half-acked batch a no-op instead of inflating actual distance and therefore a fare |
| `dispatch_sos` | repeats deliberately | re-alerts, creates no second `SOSEvent` — asserted in that direction, because if the lost alert was the first one then silence is the wrong outcome |
| `send_push_notification_task` | repeats safely | a repeated success does not take the clear-the-token branch and unregister the device; the token never reaches the logs |

---

## 14. C3–C7 — not done this run

- **C3 OTP truthfulness** — the false-success defect was fixed in an earlier run and is
  covered by `test_otp_delivery_retry.py`. Not re-verified against a live SNS failure
  this run.
- **C4 Beat topology** — beat is still embedded in the worker via `-B`. Correct for one
  worker; a second worker would double every scheduled task. Tests assert every beat
  entry names a registered task and that both money sweeps are scheduled. The topology
  decision itself is untouched. Engineering item 5 in §29.
- **C5 Redis outage matrix** — partially covered by existing suites
  (`test_driver_exclusivity_redis_down.py`, `test_sos_survives_broker_outage.py`,
  `test_driver_trip_state_unavailable.py`, `test_trip_creation_broker_coherence.py`),
  which cover exclusivity, SOS, driver trip state and booking with the broker down. The
  full six-stage matrix requested — before booking, during dispatch, after assignment,
  in progress, before completion, during stale detection — was **not** executed as one
  drill. RED rather than YELLOW, because a partial matrix is not a matrix.
- **C6 Redis restoration** — `reconcile_driver_active_trip` repairs Redis from
  PostgreSQL on reconnect and was proven on QA in the previous run's Branch A drill. Not
  re-verified this run.
- **C7 Stream recovery** — untouched.

---

## 15. Workstream D — query plans at real size (done)

Nothing had ever read a query plan. `Trip.Meta.indexes` looking complete is not
evidence: an index the planner declines, an `ORDER BY` matching nothing and sorting the
table, and a join scanning a small table inside a loop over a large one all look
identical in a model definition. And the default suite runs on SQLite with a handful of
rows per table — a size at which PostgreSQL correctly prefers a sequential scan for
everything, so no test at that size can tell a good plan from a bad one.

`qa/query_plan_audit.py` seeds a dedicated local PostgreSQL 15 database to 20,000
trips, 21,000 location points, 60 drivers and 2,000 riders, with deliberate skew — the
busiest 20% of drivers take 60% of the work, because a uniform dataset makes every
index look equally good — runs `ANALYZE`, then `EXPLAIN (ANALYZE, BUFFERS)`.

```
rider trip history                   0.32 ms   index
driver trip history                  0.15 ms   index
driver exclusivity check             0.71 ms   hash join, no large scan
operations: trips awaiting a driver  2.11 ms   index
stale active-ride sweep              9.53 ms   incremental sort over index
route replay for one trip            0.05 ms   triploc_trip_time_idx
driver location around time T        0.18 ms   triploc_driver_time_idx

No query sequentially scans a table of 5,000+ rows.
```

**A mistake in this audit is worth recording.** Its first run reported the stale sweep
sequentially scanning the whole trip table. That finding was wrong: the audit query had
been written by hand and dropped the timestamp predicates. The real query uses an
incremental sort over the `last_driver_activity_at` index. Acting on it would have
added an index nobody needed. `liveness.stale_candidates` is now split so the audit
imports the real queryset.

This is the same lesson as the fare defect from the previous run, pointing the other
way. There, a test above the real code path *missed* a defect. Here, a measurement of a
paraphrase *invented* one. Either way: audit the thing that runs.

**Two caveats, stated in the script's own output where a reader of the numbers will see
them.** These are single-query timings on one machine with a warm cache and no
competing traffic — a floor on query cost and a detector of bad plans, not a capacity
model. And the seed holds ~1,200 simultaneously-active trips where a ten-driver pilot
can have at most ten, so the sweep measurement is pessimistic by a wide margin, which
is the conservative direction.

---

## 16. D — list endpoints are bounded (done)

`REST_FRAMEWORK['DEFAULT_PAGINATION_CLASS']` is set with `PAGE_SIZE: 20`, which reads
like the question is settled. It is not: that default is applied by DRF's *generic*
views through `paginate_queryset`, and nearly every list endpoint here is a
function-based `@api_view` that builds its own response. So five tests put 120 rows
behind each endpoint and count what comes back, through HTTP with a real token.

```
rider ride-history   10 rows of 120
driver-history       10 rows of 120
notifications        20 rows of 120
driver earnings      10 rows of 120 settled trips
```

All bounded. The earnings test is worth noting: its first version asserted against an
empty feed — the fixture bulk-created trips with no ledger rows behind them — and
passed while proving nothing about the page size. It now seeds the settlement ledger and
asserts the feed is non-empty *before* asserting it is bounded.

---

## 17. D — what load testing still has not happened (RED)

This was named the largest evidence gap at the start of the run and it is still the
largest. Not done:

- 10, 25 and 50 concurrent rides through the real WebSocket protocol;
- a 100-ride simulation;
- the financial invariant under concurrent completion (one earnings row per trip,
  commission exact, no double credit) at scale — the invariant is proven for single
  rides and by `test_driver_active_trip_contention.py` for two concurrent acceptances on
  real PostgreSQL, but not under load;
- any throughput number.

**No p50/p95/p99 is reported anywhere in this document**, because nothing measured this
run supports one. The instruction was not to fake production capacity claims, and the
honest position is that this platform's capacity is unmeasured. Engineering item 3 in §29.

---

## 18. Workstream E — there is no production environment

The Railway `production` environment contains exactly one service, `SaaradhiGo-backend`:

```
state              staged-delete          (isDeleted: true)
staged change      1, STAGED since 2026-09-22, destructive: "Is Deleted" -> REMOVED
variableNames      []                     (zero)
domains            none, custom or service
volumes            none
```

There is **no** production PostgreSQL, **no** Redis, **no** Celery worker and **no** ops
console service. QA has all of them. So production is not misconfigured; it does not
exist.

---

## 19. E — production is wired to auto-deploy from `dev`

The production service's source is `repo: SaaradhiGo/SaaradhiGo-backend`, `branch:
dev`. Every push to `dev` during this run therefore triggered a build in the
**production** environment. Five such builds were triggered by this run's commits, and
every production build on record — twelve of them, going back through previous runs —
has status `FAILED`.

Nothing deployed and nothing could: with zero variables the boot guard added in an
earlier run refuses to start (`DB_HOST` must be set when `DEBUG=False`), and with no
domain nothing can reach the service. So no production deployment occurred, in any
meaningful sense. But builds *were* triggered by my pushes, and that is stated plainly
rather than described as "production was never touched".

**This is the launch-blocking part:** the instant someone gives that service its
variables and a domain, every merge to `dev` becomes a production release. The branch
wiring must change to a release branch or to manual deploys **before** production is
configured, not after. Human item 2 in §30.

---

## 20. E — production does not build the same way QA does

The production builds fail while QA's succeed, and the reason is a build-configuration
divergence rather than a code problem.

QA sets `RAILWAY_DOCKERFILE_PATH`, so it builds from the repository `dockerfile`, which
pins `python:3.12-slim`. Production has no variables, so Railway falls back to the
Railpack builder, which selects **Python 3.13**. `psycopg2-binary` publishes no cp313
wheel, so pip compiles it from source and the build fails:

```
Building wheel for psycopg2-binary (pyproject.toml): finished with status 'error'
error: failed-wheel-build-for-install
process "pip install -r requirements.txt" did not complete successfully: exit code 1
```

So even the *image* is unproven in production. Whatever eventually runs there must be
built the same way QA builds, or QA has been validating a different artifact.

---

## 21. E — dependency CVEs are blocked by the Cashfree SDK

`pip-audit` against `requirements.txt`: **14 known advisories in 2 packages.**

```
urllib3     2.0.7    PYSEC-2026-141, -1994, -1995, -1996, -1998, -1999
sentry-sdk  1.32.0   PYSEC-2026-1917
```

Neither can be upgraded, and the reason is specific: **`cashfree-pg==3.2.12` pins
`urllib3<2.1.0,>=1.25.3` and `sentry-sdk<1.33.0,>=1.32.0`.** The payment SDK is what
holds both vulnerable packages in place.

What was measured rather than assumed:

- **The upgrade is otherwise clean.** With `urllib3==2.7.0` and
  `sentry-sdk[django]==1.45.1` installed, the full suite passes: 860 passed, 9 skipped.
  So the only thing blocking it is the SDK's declared pin.
- **There is no fixed `urllib3` inside the Cashfree constraint.** Every advisory's fix
  version is above `2.1.0`. Pinning `urllib3==1.26.20` — which does satisfy the
  constraint — clears exactly one advisory of the seven and moves onto the legacy 1.26
  line. Measured, not guessed: 14 advisories become 12. That is not a fix, and it was
  not applied.
- **The SDK surface is small.** `cashfree_pg` is imported in exactly 7 places, all
  inside `servers/payments/payment_gateways/cashfree_gateway.py` plus one availability
  check in `factory.py`. So both escape routes — upgrading the SDK (3.2.12 → 4.x/5.x/6.x,
  latest is 6.0.1) or calling the Cashfree REST API directly — are tractable.

`requirements.txt` is unchanged. Both routes touch the payments path, and the standing
boundary is that a Cashfree SDK upgrade needs sandbox proof. Human item 4 in §30.

---

## 22. E — backup, restore and rollback

- **Backup:** there is no production database, so there is nothing to back up and no
  backup policy to verify. QA's PostgreSQL is a Railway-managed service; no
  backup/restore has been rehearsed for it either.
- **Restore:** never rehearsed, in any environment. An unrehearsed restore is an
  unknown, not a capability.
- **Rollback:** every production deployment reports `canRollback: false`, because none
  has ever succeeded — there is no known-good deployment to roll back to. This is a
  consequence of §18, not an independent defect, but it means the pilot currently has
  no rollback path at all.

---

## 23. E — revision visibility and boot guards (done in earlier runs, still holding)

`GET /version` reports the revision from `RAILWAY_GIT_COMMIT_SHA` and names the source
that answered, so a deploy cannot be confirmed from a stale value. Verified repeatedly
this run — it is how every QA claim in this document is anchored, and it is how the
"escalation still open on QA" statement in §0 was established.

Production refuses to boot with `DEBUG=True`, without `DJANGO_SECRET_KEY`, with
`TEST_PHONE_NUMBERS` set, with any `QA_ADMIN_BOOTSTRAP_*` present, or with
`BACKEND_URL`/`FRONTEND_URL` still pointing at QA or localhost. 26 tests, exercised as
real boots in a subprocess, each with a negative control. One of those tests is the
production boot rehearsal: it proves the required variable set is sufficient, using
dummy values.

That guard is currently the thing stopping an unconfigured production container from
serving traffic against an empty SQLite file, and §19 makes it load-bearing.

---

## 24. Error-swallowing audit — one new instance

The recurring defect pattern across this project has been a broad exception handler
turning an infrastructure failure into a confident business answer. Six instances were
found in previous runs. One new instance this run, and it is the same shape pointing at
a different system:

**Push registration wrote `None` over a working FCM token and reported success.** Not
strictly an exception handler, but the same failure: an absent input from an upstream
system (FCM had no token to give) was silently converted into a definitive business
statement (this device has no push registration). Nothing raised, nothing logged, and
the driver stopped receiving background offers. Now the absent case is distinguished
from the "unregister this device" case.

Deliberate, reviewed exceptions to the rule — infrastructure failures that are allowed
to degrade rather than refuse, each with the reasoning recorded at the code:

- the ops login guard fails **open** on a cache error (§9);
- push registration failure does not fail a login (§7);
- `record_driver_activity` swallows its own write failure rather than breaking a GPS
  ping.

Each is a case where refusing would be worse than degrading, and each says so where a
future reader will find it.

---

## 25. Boundary tests added this run

Every feature changed this run has at least one test at the highest practical consumer
boundary.

| Change | Boundary | Tests |
|---|---|---|
| Operator privilege boundary | HTTP, real JWT, direct endpoint calls | 41 |
| Push registration | HTTP, real two-step OTP flow | 11 |
| Celery worker loss | Real broker + real worker subprocess + real SIGKILL | 14 assertions |
| Duplicate execution | Celery task machinery + real database | 12 |
| List pagination | HTTP, real token, 120 rows behind each endpoint | 5 |
| `visibility_timeout` | Configuration, with the derivation pinned | 3 |
| Query plans | Real PostgreSQL 15 at 20k trips | harness |

Two of these were verified by running them against the pre-fix code — 11 of 41 and 6 of
11 fail there. That check is the difference between a regression lock and a test that
would have passed either way, and it is worth making a habit.

---

## 26. Financial boundaries — all held

Nothing in this run enabled `final_fare`, enabled actual metered billing, activated a
promo, changed GST, added a cancellation fee, invented an abandoned-ride charge,
invented a Cashfree status, or automatically refunded an unresolved payout.

Positively asserted, not merely avoided:

- `compute_trip_actuals` never writes `final_fare`, however many times it runs (§13);
- the Branch B drill observed `final_fare = None` after 12 minutes of silence on an
  `in_progress` ride (§3);
- `test_final_fare_is_not_in_the_quote` holds that a quote never asserts a metered
  final fare;
- the duplicate-execution suite asserts no second receipt, no second earnings row and
  no duplicated GPS point — the last of which matters because duplicated points inflate
  actual distance, which feeds a fare.

---

## 27. Launch scorecard

GREEN / YELLOW / RED / BLOCKED. No numeric score — a number would average away the two
items that decide this.

### Operations

| # | Item | Status | Evidence |
|---|---|---|---|
| 1 | QA operator account exists | **BLOCKED** | Command shipped and tested (16 tests); Railway CLI unauthenticated and QA bootstrap password unreadable, so it has never been run. |
| 2 | Operator login rehearsal | **BLOCKED** | Depends on item 1; no operator workflow has been performed by a human or by me. |
| 3 | Driver onboarding via console | **BLOCKED** | Depends on item 1; the ONBOARD link in the lifecycle chain remains unexercised. |
| 4 | KYC approval workflow | **BLOCKED** | Depends on item 1; authorization on the endpoint is now proven (§5) but the workflow is not. |
| 5 | Ride lookup and active-ride view | **BLOCKED** | Depends on item 1. |
| 6 | Stale-ride detection | **GREEN** | Branch B drill on QA: 12 minutes of silence, no false auto-cancel, money untouched, PostgreSQL authoritative, 5/5. |
| 7 | Stale-ride operator queue | **YELLOW** | 51 tests including negative controls; the queue page itself has never been opened because of item 1. |
| 8 | Audited recovery actions | **YELLOW** | Tested end to end including the audit row; never exercised through the console. |
| 9 | SOS operator path | **BLOCKED** | SOS creation, dedup and dispatch are proven; the operator's side of it is not. |
| 10 | Support tooling + IDOR | **RED** | Not tested this run. Cross-user access on support endpoints is unverified. |
| 11 | Financial investigation from ops UI | **BLOCKED** | Depends on item 1. |
| 12 | Unresolved payout presentation | **YELLOW** | `unresolved` (not `FAILED`) is implemented and tested; the UI has not been seen. |

### Security and notifications

| # | Item | Status | Evidence |
|---|---|---|---|
| 13 | Operator privilege boundary | **GREEN** | Escalation demonstrated, then closed by two independent fixes; 41 tests, 11 of which fail pre-fix; verified closed on the running QA deployment at `8181e2c`. |
| 14 | Sensitive-action authorization via direct API | **GREEN** | Every operator endpoint refuses rider, driver, anonymous and role-without-staff callers, asserted by direct calls with real tokens. |
| 15 | Ops login brute force | **GREEN** | Was 12 failures in 7.8 s unthrottled; now bounded per phone and per IP with a non-enumerating refusal, 10 HTTP-boundary tests. |
| 16 | Ops console MFA | **RED** | Not started. The console approves KYC and releases payouts behind a password alone. |
| 17 | RBAC granularity | **RED** | There is no support role; every operator is a full admin with payout and KYC authority (§6). |
| 18 | Driver FCM registration | **GREEN** | Three defects fixed including a silent token wipe; 11 HTTP tests, 6 of which fail pre-fix. |
| 19 | FCM token rotation | **RED** | No `onTokenRefresh` listener; a long-signed-in driver can hold a token the server does not have. |
| 20 | Background ride offer | **YELLOW** | Server path tested, client handler present and degrades quietly; no real device has been observed receiving one. |
| 21 | Rider notification audit | **RED** | Not performed this run. |
| 22 | Credential hygiene | **GREEN** | No credential printed by any harness; token-absence-from-logs asserted on success and failure paths. |
| 23 | Maps API key restriction | **RED** | No restricted, platform-bound key exists. |
| 24 | Push token as a credential | **GREEN** | Now write-only in the serializer; verified no client reads it back. |

### Celery and Redis

| # | Item | Status | Evidence |
|---|---|---|---|
| 25 | Worker-loss durability | **GREEN** | Real SIGKILL against a real broker: message survives, is redelivered, and completes. 14/14. |
| 26 | Lost-task recovery time | **YELLOW** | Now bounded at ~15–17 min by a derived `visibility_timeout` of 900 s. Bounded and measured, but slow for a 90-second cancellation deadline. |
| 27 | Duplicate-execution safety | **GREEN** | All six ride/money/safety tasks executed twice; no second receipt, notification, SOS event or GPS point. |
| 28 | Worker platform constraint | **GREEN** | A Windows worker never recovers a dead worker's tasks; documented, and the drill prints its platform so a Windows pass cannot be misread. |
| 29 | OTP delivery truthfulness | **YELLOW** | Fixed and tested in an earlier run; not re-verified against a live provider failure. |
| 30 | Celery Beat topology | **YELLOW** | Correct for one worker; embedded `-B` would double every scheduled task at two. Beat entries are validated against the registry. |
| 31 | Redis outage matrix | **RED** | Four partial suites exist; the six-stage matrix was not executed as one drill. |
| 32 | Redis restoration from PostgreSQL | **YELLOW** | Proven on QA in the previous run; not re-verified this run. |
| 33 | GPS stream recovery | **RED** | Untouched. |

### Load, performance, production

| # | Item | Status | Evidence |
|---|---|---|---|
| 34 | Query plans at scale | **GREEN** | 20,000 trips on real PostgreSQL 15; no query sequentially scans a table of 5,000+ rows. |
| 35 | List endpoint bounds | **GREEN** | 120 rows behind each endpoint; every one returns a page, verified through HTTP. |
| 36 | Measured concurrency and throughput | **RED** | No concurrent-ride load test, no 100-ride simulation, no percentile. Deliberately no number is quoted. |
| 37 | Production environment | **BLOCKED** | One service, marked deleted, zero variables, no domain, no database, no Redis, no worker, no console. |
| 38 | Production release safety | **RED** | Production builds from `dev`; twelve builds triggered, all failed, and they build with a different builder and Python version than QA. |

---

## 28. The nine questions

**1. Can a rider complete a ride end to end without an engineer? — YES.**
Proven repeatedly on QA: acceptance ride 30/30 on trip 46, and the abandoned-driver
Branch A recovery on trip 47 where a crashed driver reconnected, recovered the trip,
completed it, confirmed cash and returned to supply with no human involved.
*Minimum remaining condition: none for the happy path.*

**2. Is the money correct? — YES, for cash rides at pilot scale.**
169.02 gross − 33.80 commission = 135.22 net, exactly; one earnings row per trip; no
trip anywhere in the feed with more than one; `final_fare` NULL; the displayed
decomposition closes to the paisa at the HTTP boundary.
*Minimum remaining condition: the same reconciliation performed after a concurrent-load
run, so the invariant is known to hold under contention and not only in sequence.*

**3. Can an abandoned ride be recovered without an engineer? — YES for the driver's
return, NOT YET for the operator's decision.**
Branch A is proven on QA with no human. Branch B is now proven on QA for the three
things that must *not* happen — no false cancel, no money moved, PostgreSQL
authoritative — but the operator side of it has never been performed.
*Minimum remaining condition: item 1 of the scorecard, then one timed Branch B where a
human resolves trip 49 from `/stale-rides/`.*

**4. Is work lost when a worker dies? — NO.**
Real SIGKILL, real broker: the message survives, is redelivered, and completes. Every
redeliverable task is proven safe to run twice.
*Minimum remaining condition: none for correctness. For latency, see question 5.*

**5. Is the recovery fast enough to be invisible to a rider? — NOT YET.**
Recovery is `visibility_timeout` plus up to ~100 s, so ~15–17 minutes. A rider whose
`auto_cancel_trip` was on the dead worker waits that long past a 90-second deadline.
*Minimum remaining condition: either accept and document a 17-minute worst case as pilot
policy, or move the accept-timeout deadline off the broker's redelivery path — a
database-backed sweep for trips stuck in `requested` past their timeout would bound it
at the sweep interval instead.*

**6. Can an outsider gain operator authority? — NO, now.**
Two independent defects that together allowed it are closed, with 41 tests and a
demonstrated pre-fix failure.
Verified closed on the running QA deployment at `8181e2c`: `role=admin` is refused with
`AUTH_INVALID_ROLE` while `role=rider` still succeeds.
*Minimum remaining condition: the console needs MFA before real operators use it, since
a password is currently its only gate.*

**7. Is the operations console fit for a real operator? — NOT YET.**
Brute-force protection is in place and the authorization boundary is now sound, but
nobody has ever signed in, there is no MFA, and there is no role separation between
reading a trip and releasing a payout.
*Minimum remaining condition: item 1 of the scorecard, plus MFA. RBAC can follow if the
pilot runs with one or two trusted operators, provided that is a stated decision.*

**8. Do we know what this platform can carry? — NO.**
Query plans are healthy at 20,000 trips and every list endpoint is bounded. Nothing
else about capacity has been measured, and no percentile is claimed anywhere in this
report.
*Minimum remaining condition: one concurrent-ride harness at 10, 25 and 50 with the
financial invariant checked afterwards. Until then, capacity statements should not be
made at all.*

**9. Can we deploy to production? — NO.**
There is no production environment to deploy to, it builds differently from QA, it has
never had a successful deployment, there is no rollback target, and no restore has ever
been rehearsed. Separately, it auto-builds from `dev`.
*Minimum remaining condition: change the deploy trigger off `dev` FIRST; then provision
PostgreSQL, Redis, worker and console; set the variable set the boot rehearsal proves
sufficient; build with the Dockerfile as QA does; take and restore one backup before
any real rider exists.*

---

## 29. Next five — engineering

Risk ordered.

1. **Close the FCM token-rotation gap.**
   Add an `onTokenRefresh` listener in the driver app and a narrow endpoint for it to
   call, so a driver who stays signed in does not silently end up holding a token the
   server no longer has. *Why first: it is the same silent-push failure just fixed in
   §7, arriving by a different route — and it fails exactly the same way, with nothing
   raised and nothing an operator can see.*

2. **Ops MFA, then the minimum RBAC split.**
   MFA with an established TOTP library — no invented cryptography. Then the smallest
   honest role split: a role that can read trips, drivers and rides and act on stale
   rides, and cannot approve KYC or release a payout, enforced in
   `base.permissions`/`is_operator` where the boundary already lives. *Why second: the
   console's only gate on payout release is a password, and every operator currently has
   every authority.*

3. **The concurrent-ride load harness.**
   10, 25 and 50 concurrent rides through the real WebSocket protocol against a real
   stack, then the financial reconciliation over the result: one earnings row per trip,
   commission exact to the paisa, no double credit, no duplicated GPS point. Report a
   percentile only if the methodology supports one. *Why third: it is the largest
   remaining evidence gap and it is the one that would expose a contention defect before
   a rider does.*

4. **The Redis outage matrix, as one drill.**
   All six stages in sequence — before booking, during dispatch, after assignment, in
   progress, before completion, during stale detection — with the restoration step at
   the end. *Why fourth: four partial suites exist and each is sound, but nobody has
   watched a ride survive Redis disappearing at each point of its life.*

5. **Move the accept-timeout deadline off broker redelivery, and split beat out.**
   A database-backed sweep for trips stuck in `requested` past their timeout bounds the
   worst case at the sweep interval instead of 17 minutes, and answers question 5
   properly. In the same change, move beat out of the worker so a second worker becomes
   possible without doubling every scheduled task. *Why fifth: both are known, bounded
   and currently survivable; they become urgent the moment there is more than one
   worker.*

---

## 30. Next five — human and external

Risk ordered.

1. **Run `bootstrap_qa_operator` against QA.**
   ```
   railway run --service backend --environment QA \
       python manage.py bootstrap_qa_operator
   ```
   The three variables are already set in QA; their values are not knowable from this
   environment and are not needed to ship the command. Then hand over the operator
   credential by whatever channel you use for secrets. *Why first: it single-handedly
   unblocks seven scorecard items and four of the nine questions. Nothing else on either
   list unlocks as much.*

2. **Change the production service's deploy trigger off `dev` — before configuring
   anything else there.**
   Either point it at a release branch or set it to manual deploys. Also resolve the
   destructive staged change that has been pending against it since 2026-09-22: either
   commit the deletion deliberately, or discard it. *Why second: today those builds fail
   harmlessly. The moment the service has variables and a domain, every merge to `dev`
   ships to production, and that is a configuration that produces an accident rather
   than a decision.*

3. **Watch a real handset receive a backgrounded ride offer.**
   A physical Android device, a real Firebase project, the driver app installed, screen
   off, app backgrounded, and a dispatch sent. Nothing in a test suite can substitute
   for this, which is why item 20 is YELLOW and not GREEN. *Why third: push is the
   fallback dispatch channel, and whether it works has never been observed.*

4. **Decide the Cashfree SDK path, in the sandbox.**
   `cashfree-pg==3.2.12` is holding 14 CVEs in place, and there is no fixed `urllib3`
   inside its constraint. Either upgrade the SDK (latest 6.0.1) with sandbox proof of
   order creation, payment status and refund, or replace its 7 call sites with direct
   REST calls. Both are tractable; neither is safe to do without a sandbox run. *Why
   fourth: it is the only thing blocking a straightforward security upgrade that the
   test suite already shows is otherwise clean.*

5. **Provision production and rehearse a restore before any rider exists.**
   PostgreSQL, Redis, the Celery worker and the ops console; the variable set the
   production boot rehearsal proves sufficient; `RAILWAY_DOCKERFILE_PATH` so it builds
   the same artifact QA validates. Then take a backup, restore it somewhere, and confirm
   a trip survives the round trip. *Why fifth: it depends on item 2 being done first,
   and an unrehearsed restore is not a backup policy — it is an assumption.*

---

## 32. Workstream F — final pilot proof

F cannot be completed, and the reason is structural rather than a matter of remaining
effort: a *final pilot proof* is a statement about a system running where the pilot will
run. There is no such system (§18), and the operator half of the product has never been
exercised by a person (§2). Anything called a final proof today would be a proof about
QA, described as a proof about production.

What *is* established, and would form the body of that proof once A and E are closed:

| Claim | Status |
|---|---|
| A rider can book, ride and pay end to end, no engineer | Proven on QA, 30/30, trip 46 |
| The money reconciles exactly, one earnings row, `final_fare` NULL | Proven on QA, trip 46 |
| The displayed fare decomposition closes to the paisa | Proven at the HTTP boundary |
| A crashed driver recovers their own trip and returns to supply | Proven on QA, trip 47 |
| An abandoned ride is neither cancelled nor charged | Proven on QA, trip 49, 12 min silence |
| A killed worker loses no work | Proven, real SIGKILL, real broker |
| Every redeliverable task is safe to run twice | Proven, 12 tests |
| No dominant query scans a large table at 20k trips | Proven, real PostgreSQL 15 |
| An outsider cannot gain operator authority | Proven, and verified on the QA deployment |
| Nobody has signed in to the operations console | **Not proven — never attempted** |
| The platform's capacity | **Unmeasured** |
| Production runs, rolls back and restores | **Does not exist** |

The first nine lines are a genuine pilot-mechanics proof and they are worth reading as
one. The last three are why this section does not claim more.

---

## 33. Honest summary

The pilot's *mechanics* are in good shape and getting better: a ride completes, the
money reconciles to the paisa, a crashed driver recovers themselves, an abandoned ride
is neither cancelled nor charged, a killed worker loses no work, every redeliverable
task is safe to run twice, and no dominant query scans a large table.

The pilot's *operations and production* are not. Nobody has ever signed in to the
console, so the entire operator half of the product is unrehearsed. Capacity is
unmeasured and this report quotes no capacity number. Production does not exist, builds
differently from QA, has never deployed successfully, has no rollback target, and is
wired to release on every push to `dev`.

This run also found that anyone with a phone could have become an operator. That is
fixed, but it is worth sitting with: the defect was two years of plausible-looking code
plus a docstring describing a check that was never written. It was found by calling the
endpoint as an attacker would rather than by reading the permission class. That is the
same lesson as the fare defect and the query-plan false alarm, and it is the one habit
this project should keep: **test at the boundary a real consumer uses, and verify the
test fails without the fix.**

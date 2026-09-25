# India pilot — final blockers

Written for: whoever decides whether this pilot launches, and the two or three people who
will clear what is left.

**Backend `b6be1e8` on `dev`** · QA verified at `a640e1b` through `/version`
**1105 backend tests passed, 9 skipped** · `ruff` clean · no pending migrations
**Rider and driver CI green, both producing APK artifacts**

The goal of this run was to turn the remaining RED and BLOCKED items into a short,
finite checklist. It is ten items long, and **six of the ten are not engineering work.**

---

## Read this first

**One security defect found this run was serious enough to change the launch
conversation.** Anyone able to receive an SMS could make themselves a platform operator:
one OTP request carrying `"role": "admin"`, one login, and the resulting account read the
driver KYC queue, the payout queue, every trip, and every driver's live position. All
HTTP 200, on an account with `is_staff=False`.

It was found by attacking the API the way an attacker would, not by reading the
permission class — the class's own docstring described a check that had never been
written. It is closed, verified closed on the running QA deployment, and held by 60
tests that attack it from every path the architecture offers.

**The pilot is RED, and it is held there by three things, two of which are not code:**
an unrotated database credential, no operator account, and therefore no exercised driver
onboarding.

---

## Operator acceptance

**BLOCKED.** No operator account exists, so not one operator workflow has been performed
by a person.

`bootstrap_qa_operator` is built, tested (16 tests), refuses to run outside QA, refuses a
password under 12 characters, is idempotent, never prints a credential, and audits
itself without recording a phone number. It has never been run.

Two independent paths confirm why: the Railway CLI is unauthenticated (`whoami` exits 1
with no output), and the Railway MCP server is authenticated but returns
`valuesRedacted: true` — variable names only. `QA_ADMIN_BOOTSTRAP_PHONE`,
`QA_ADMIN_BOOTSTRAP_CODE` and `QA_ADMIN_BOOTSTRAP_PASSWORD` are confirmed **present** in
QA. Their values are not readable from here, and no MCP tool runs a command in a service.

I also could not detect whether an operator had been created since the last run, and the
reason is a property I built deliberately: the console login returns an identical refusal
for a wrong password, an unknown phone and a locked account, so it cannot be used to
discover which accounts exist. The non-enumeration guard works, and it works against me
too.

**This blocked showed up in practice within the hour.** The Branch B stale-ride drill
deliberately leaves its trip `in_progress` for an operator to resolve. With no operator,
trip 49 stayed in progress, its driver stayed out of supply, and the next acceptance ride
was refused with "You are already on an active ride." The operator gap is not
theoretical.

---

## Driver onboarding

**BLOCKED**, behind the operator account.

The driver's own half works and is proven: registration, authentication, going online,
receiving an offer, completing a ride, cash confirmation, earnings. The 30-stage
acceptance ride exercises all of it.

What has never happened is the operator's half — finding a pending driver, reviewing
their documents, approving them. The authorization on those endpoints is now proven
correct (a driver cannot approve their own KYC; a support-scoped operator cannot either),
but authorization is not a workflow.

---

## Admin escalation regression

**RESOLVED, and re-attacked from every direction.**

```
OTP step, 18 role spellings (casings, whitespace, injections, wrong types)   refused
role in the LOGIN body                                     proven inert, not assumed
JWT with a forged but VALIDLY SIGNED role/is_staff claim                     ignored
expired token                                                               refused
six malformed tokens, including alg:none claiming admin                     refused
the separate /auth/admin/login/ password endpoint                     three defects
rider token, driver token, anonymous, against six privileged endpoints      refused
a legitimate operator, and a superuser                                     ALLOWED
```

That last line matters as much as the rest. A suite that only proves refusals is
satisfied by an outage.

**There were five independent definitions of "is this an operator", not one.** An
attacker needs only the most permissive, and nobody notices the others are stricter:

1. `IsAdmin` — role OR superuser, no `is_staff`. This is what the escalation used.
2. `admin_required` on the console — role OR superuser.
3. the console login view — role OR superuser.
4. `auth_user.admin_views.admin_login` — role OR `is_staff` OR superuser, on a
   **token-minting** endpoint, so a rider carrying `is_staff` could mint an operator JWT.
5. `pricing.permissions.IsPlatformAdmin` — role OR `is_staff`, guarding the **rate
   card**, which is the number a rider is charged.

There is now one definition, and a test asserts the console imports the same object
rather than a copy.

That fourth endpoint had two further defects: it was **completely unthrottled** (no
`throttle_classes`, and the project sets throttle *rates* without throttle *classes*), and
it **enumerated accounts** — unknown phone and wrong password returned distinguishable
401s, and a third distinct 403 identified which accounts were operators. All four failure
modes are now one identical refusal, asserted by comparing response shapes for equality.

---

## MFA

**YELLOW — complete in code, unenrolled in fact.**

Built on `django-otp`: TOTP per RFC 6238 and single-use static recovery codes, with the
cryptography done by the library. Nothing here implements an algorithm.

The design decision that makes it real rather than decorative: **MFA is enforced where
the credential is ISSUED, not where it is used.** `/auth/admin/login/` will not mint a
token without a code, so an operator token cannot exist without two factors behind it,
and every downstream authorization check can trust it without knowing MFA exists. Had the
token been issued first and the factor demanded only by the console, POSTing straight to
the payout endpoint would have skipped it.

Three bypasses closed and tested:

- **Straight at the API** — no token without a code.
- **Via the SMS path** — `/auth/login/` trades an SMS code for a token with no password
  and no second factor, and that token is the stronger credential of the two. An operator
  using it would have made MFA cost one SIM swap. Operators are refused there, with the
  ordinary invalid-OTP shape so the refusal does not identify operator phone numbers.
  Riders and drivers are unaffected, and two tests hold that.
- **Session cookie alone** — a signed-in session is one factor; every console page
  bounces to the challenge until this session has presented a code.

Production enforces it with **no override** — matching the boot guards, where a guard
that can be satisfied by setting a flag is a comment. An operator with no enrolled factor
cannot sign in under enforcement, which is what makes enrolment happen rather than being
deferred.

Recovery, which is the real risk with one or two operators: ten single-use codes at
enrolment, and `manage.py reset_operator_mfa` requiring `--confirm` and shell access.
That bar is deliberate — any self-service MFA reset is by construction a way to take over
the account it protects, and a reset link to an operator's email moves the whole security
boundary onto that mailbox. Reset does not leave the account exempt.

**The gate stays YELLOW because no operator has enrolled, because no operator exists.**

Two things learned while writing it, both recorded at the code: a numeric JSON
`mfa_code` 500'd the view before it was coerced, and `django-otp` throttles per *device*
as well — a failed attempt refuses the next one even when valid. That second one is an
extra layer and is now asserted as a property, with the operational consequence noted:
an operator who fumbles a code has to wait, not retry immediately.

---

## Rate limiting

**GREEN on both operator sign-in surfaces.**

The console form was measured at 12 consecutive failed logins in 7.8 seconds before its
guard existed. The API password endpoint had exactly that exposure and **no guard at
all** — two surfaces with different brute-force resistance means the weaker one is the
real policy. They now share one guard.

Properties held by tests: repeated failures refuse; the refusal is indistinguishable from
an ordinary wrong password, so a lockout cannot be used to find real accounts; an unknown
phone looks identical to a known one; **a correct password on a non-operator account
still counts as a failure**, or every rider account is a free password-guessing oracle;
the lockout is bounded in time; a per-IP bound catches one host working through a list;
and the MFA challenge is rate-limited too, because six digits inside a 30-second window
is a short brute force against an already-authenticated session.

The guard **fails open** on a cache error. Deliberate, and asserted so it is a recorded
choice: the cache is Redis, and locking every operator out during a Redis incident is
worse for a ten-driver pilot than temporarily losing brute-force protection.

---

## RBAC

**GREEN, minimally.**

There was no support role — `role` had three values, so every operator could approve KYC,
release payouts and rewrite the rate card. Four operator roles now, nine capabilities,
one flat mapping. No permission editor, no per-object rules, no inheritance: an
authorization system nobody can reason about is worse than the coarse one it replaced.

```
support     trip.read driver.read support.read support.reply
driver_ops  trip.read driver.read driver.kyc support.read
finance     trip.read driver.read support.read finance.read finance.payout
admin       everything
```

Flat rather than nested on purpose — a hierarchy would make finance a superset of support
by accident of ordering rather than by decision.

**§6's stated requirement, asserted by direct API call: a support account cannot approve
a payout.** Six tests fail if the capability gates are replaced with the old blanket
check, which is what makes the separation load-bearing rather than decorative. The
controls matter equally: finance can still approve payouts, driver_ops can still approve
KYC, support can still reply to tickets, and every operator can still read trips and the
dashboard.

`ops_role` grants nothing on its own — it is only consulted for accounts that are already
operators — because a new authority field is exactly the sort of thing that becomes the
next escalation route.

The migration backfills every existing row to `admin` deliberately. Every row the field
can affect already holds every authority today, so backfilling least-privilege would
silently strip payout authority from whatever operator accounts exist — an outage dressed
as a security improvement.

---

## Audit

**GREEN.** Actor, actor label that survives account deletion, action, target type and id,
before/after state, reason, IP, user agent, timestamp.

Covered: KYC approve and update, driver deletion, withdrawal approve and reject and bulk
action, DPDP erasure, QA operator bootstrap, MFA enrolment, MFA reset, operator API
sign-in, console MFA verification, and **pricing** — which was the one privileged action
with no trail at all, so a fare dispute could not be answered with who changed what and
when. SOS transitions keep their own immutable `SOSEventUpdate` trail with actor, status
and note.

No unnecessary PII: no phone number, no TOTP secret, no recovery code, no FCM token in
any audit row — asserted, including on failure paths.

The pricing audit is guarded twice, and the second guard came from a test. The recorder
swallows its own exceptions so the mixin looked safe; a test that replaced the recorder
with a raising stub produced a 500 on the pricing change, which is exactly what the
docstring promised could not happen. A promise about the caller has to be kept by the
caller.

---

## Driver FCM

**GREEN for registration. RED for rotation.**

Three defects fixed, all on the most-travelled path in the product:

1. **Any login without a device token erased the stored one.** The assignment was
   unconditional and `device_token` is optional. The driver app documents four ordinary
   states in which it has no token to send — no Firebase config, no Play Services,
   permission not granted, no network at launch — and omits the field in each. A driver
   who hit one of them lost push, silently, until some later login happened to carry a
   working token. Push is the fallback dispatch channel, so the failure produces a driver
   who stops getting background offers and cannot say why.
2. **A bare `save()` wrote every column** from a possibly-stale instance — the same shape
   as the defect that used to revert accepted trips.
3. **The token was echoed back in every user payload**, including any admin view of a
   driver's profile. An FCM token is a device credential. Now write-only, after checking
   all three clients: none reads it.

11 HTTP-boundary tests, **6 of which fail against the pre-fix code.**

**Still open:** the app registers a token at login only. FCM rotates tokens, and there is
no `onTokenRefresh` listener, so a driver signed in for months can hold a token the
server does not have. That needs a mobile change and a refresh endpoint.

---

## Background offer

**YELLOW, and it must stay YELLOW.**

The server path is tested and the client handler exists and degrades quietly. **No real
device has been observed receiving a backgrounded ride offer.** That requires a physical
handset, a real Firebase project and someone watching it. Nothing in a test suite
substitutes for it, and marking this GREEN from unit tests would be exactly the kind of
claim this report exists to avoid.

---

## Celery recovery

**GREEN**, with the rider-visible case specifically fixed.

The kill drill measured it: a task whose worker dies is redelivered after
`visibility_timeout` plus up to ~100 s of restore-poll granularity — roughly 15-17
minutes at 900 s. That is the default recovery objective, and exactly one task could not
live with it.

`auto_cancel_trip` is scheduled once with a 90-second countdown, and that single deadline
bounds the whole driver search and produces the frame telling a rider nobody accepted.
Fifteen minutes of silence there is the failure: the rider watches a search already given
up on and cannot rebook.

**`visibility_timeout` was not lowered.** Its floor is the longest legitimate unacked
wait — a 180 s countdown plus a 360 s execution — and below that a second worker takes a
message the first is still working on. One setting cannot serve a 90-second deadline and a
six-minute task.

So that deadline got its own path, in the database: `ride.sweep_unaccepted_trips` runs
from beat every two minutes and reaches the decision by calling `auto_cancel_trip`
directly, not through the broker. **~3 minutes instead of ~17.** Most of its 14 tests are
negative controls, because those are what matter: accepted, in-progress, completed,
inside-deadline and inside-grace trips are all untouched, no money moves, the sweep is
bounded, one wedged row does not stop the rest, and a late redelivered message finds
nothing to do and does not rewrite `cancelled_at`.

All nine tasks are classified by recovery objective, and the classification is a **test**
rather than a table in a runbook, so it fails when it drifts. Queue separation was
considered and deliberately not done, with the trigger recorded: the load run measured a
Celery backlog of 0 through 100 concurrent rides.

---

## Redis recovery

**GREEN.** All seven stages, 22 tests: before booking, during dispatch, after assignment,
in progress, before completion, during stale detection, and recovery.

Two things asserted at every stage: PostgreSQL still holds the truth, and nothing
fabricates a business answer out of an infrastructure failure. The second is this
codebase's recurring defect — seven instances found across runs, each a broad handler
turning "I don't know" into a confident answer.

The most important line: **whether a driver is busy is answered from PostgreSQL**, so a
Redis outage cannot make a busy driver look free and let a second rider be assigned the
same driver. And when Redis returns it is repaired from PostgreSQL in both directions,
with no shell and no operator — which is the trip-42 failure closed at its root.

Recorded honestly: this does **not** cover the channel layer. Redis also backs Channels,
so a real outage also stops WebSocket fan-out and a rider mid-ride stops receiving
updates. These tests are about state and money surviving, not about frames arriving.

---

## Load test

**GREEN**, against an isolated stack — its own PostgreSQL, Redis, Daphne, Celery worker
and Celery beat on private ports with a throwaway database, built from the Dockerfile on
Python 3.12, the same artifact QA validates. Never shared QA: a hundred synthetic rides
would pollute the tables every money reconciliation reads.

```
stage                  attempted  completed   OTP push missed   recovered by pull
10 concurrent                 10         10                 0                   -
25 concurrent                 25         25                 0                   -
50 concurrent                 50         50                 6                   6
100, batches of 25           100        100                 2                   2
30 riders / 10 drivers        30         10 (by design)      -                   -
```

Latency from the 100-ride run, client-side, frame sent to the frame answering it (ms):

```
                     n     min  median     p95     p99     max
  booking          100      31     205     691     801     830
  command.accept   100      33     220     395     454     490
  command.reached  100      26     316     555     580     585
  command.start    100      27     422     660     718     730
  command.complete 100      53     284     612     716     874
  confirm_cash     100      49     462     978    1064    1094
  dispatch          99      24     377    1319      --    1436
```

p95 is printed only with ≥20 samples and p99 only with ≥100 — below that a percentile is
arithmetic pretending to be evidence, and the harness says so in its own output.

**This is not a production capacity model and no capacity number appears in any of these
documents.** One machine, localhost networking, no TLS, no mobile radio, a warm cache and
`fsync=off` — every one of those flatters the result. S3, FCM, SMS and Cashfree are absent
from the stack, which exercises the product's degradation paths rather than the providers.

---

## 100-ride simulation

**GREEN.** 100 of 100 completed in batches of 25, in 65 seconds of wall clock, with
`rejected_connections: 0` and a Celery backlog of 0 before and after.

### The finding it produced

At 50 concurrent, **6 of 50 riders never received the accept-time OTP on either
socket** — not the trip channel, not their personal channel. It was not a missing
subscription in my harness: I added the rider's trip channel, which a real app opens, and
it stayed at 5-6.

The fan-out is best-effort by design. `channels_redis` drops to a channel whose queue is
full and `group_send` swallows that per channel, so a client not draining continuously
misses the frame.

All 6 recovered through `GET /ride/active/`, which returns the OTP to the rider on that
trip. **The product is recoverable because the pull path exists** — but the rider app only
read it on startup and resume, so a rider who missed the push saw `----` where their OTP
should be, with a driver waiting outside, until they backgrounded the app and reopened it.
Twelve percent of riders at fifty concurrent rides.

The rider app now refetches once when it reaches a state that should have an OTP and does
not. Asserted in both directions, because a fallback firing on every accept would add a
round trip to every ride.

---

## Financial invariants

**GREEN at every load stage**, reconciled over the generated dataset rather than asserted
per ride:

```
every completed trip settled exactly once      no unsettled, no duplicates
no duplicate wallet movement                   idempotency keys unique
no trip carries more than one settlement row
final_fare NULL on every trip
commission exact to the paisa                  against each trip's own recorded fare
no driver ever held two overlapping trips
```

Financial boundaries held: nothing enabled `final_fare`, metered billing, promos, GST
changes, cancellation fees, an invented abandoned-ride charge, an invented Cashfree
status, or an automatic refund of an unresolved payout. Positively asserted, not merely
avoided — `compute_trip_actuals` never writes `final_fare` however many times it runs, and
the recovery of abandoned trip 49 on QA left `final_fare` None.

---

## Driver exclusivity

**GREEN under deliberate contention.** 30 concurrent riders against 10 drivers: exactly
10 accepted, 20 refused, zero overlapping trips. The refusals were deterministic
rejections, not errors and not double assignments — the invariant holding under real
contention rather than under a test that hoped for it.

---

## Mobile build artifacts

**GREEN.** Both apps: `analyze-and-test` and `build-android` green, artifacts produced
from current HEAD (rider 95 MB, driver 89 MB).

One honest note. My rider commit initially failed the analyze gate on two unused imports
in my own test file. I had checked locally with a grep requiring spaces around the word
"warning" while the real output format is `warning •` at line start, so the grep returned
zero and I read it as clean. Fixed by running the exact CI command and checking its exit
code instead of pattern-matching its output. The APK job itself never failed.

---

## Production architecture, source strategy, environment

All three: **PREPARED, NOT APPLIED.** See `india-pilot-production-package.md` for the
topology, the release strategy and the variable-name checklist.

The three facts that matter here:

- **Production does not exist.** One service, marked deleted, with a destructive staged
  change pending since 2026-09-22, zero variables, no domain, no database, no Redis, no
  worker.
- **It builds from `dev`**, so every push this run triggered a production build. All
  twelve on record failed. Harmless today; the moment production has variables and a
  domain, every merge becomes a release.
- **It would not build the same artifact QA validates.** With no variables it falls back
  to Railpack on Python 3.13 and dies compiling `psycopg2-binary`, which has no cp313
  wheel. QA builds the Dockerfile on 3.12.

---

## Backup and restore

**RED.** Never rehearsed in any environment. Railway provides automated PostgreSQL
backups; that is not a backup policy, and a backup nobody has restored is an assumption.
The drill and its acceptance criterion are in the production package: restore into a
scratch database, point a non-production backend at it, and confirm a known completed trip
reads back with its fare, settlement row and receipt. Record the time — that number is
the recovery objective.

---

## S3

**RED.** Requirements written, nothing verified against a real bucket: private buckets,
separate per environment, presigned access only, least-privilege credential scoped to one
prefix, server-side encryption, KYC and receipts under distinct prefixes. No public KYC is
a requirement, not yet a fact.

---

## Secrets

**RED, and this is the one that holds the gate regardless of everything else.**

A database credential appeared in the repository's history. **Removing a credential from
HEAD is not rotation** — clones, forks, CI caches and local checkouts all still have it.
The narrow question for a human with Railway access: *is that credential still valid for
any database that exists today?* If yes, production is RED whatever the application
readiness says, and rotation is the first action. If no, record when it was rotated or
when the database was recreated, and close the item.

This is a credential operation, not an architectural defect, and it should not be
reported as one.

---

## Maps

**RED.** One unrestricted key. Required before either app reaches a store: an Android key
restricted to the package name and release signing SHA-1, an iOS key restricted to the
bundle identifier, a server key restricted by IP and by API — three keys, not one shared
key, because a key embedded in a shipped APK is extractable by anyone who downloads it.
Plus a billing quota and an alert, which converts a leaked key from a five-figure bill
into a broken feature.

---

## Cashfree

**OFF, intended.** Online payments off, automatic payouts off, cash pilot.

No SDK upgrade was attempted, and specifically not to clear CVEs. The sandbox validation
plan is in the production package: create, duplicate `transferId`, timeout, status
lookup, pending, success, failure, provider reference. The application side already
handles the hard part — an ambiguous provider answer becomes `unresolved`, is never shown
as `FAILED`, and offers no blind Retry.

---

## Dependencies

**RED, and blocked by one pin.** 14 advisories across 2 packages:

```
urllib3     2.0.7    PYSEC-2026-141, -1994, -1995, -1996, -1998, -1999
sentry-sdk  1.32.0   PYSEC-2026-1917

cashfree-pg==3.2.12   requires urllib3<2.1.0,>=1.25.3
                      requires sentry-sdk<1.33.0,>=1.32.0
```

Measured rather than assumed: the upgrade is otherwise clean (the full suite passes with
`urllib3==2.7.0` and `sentry-sdk==1.45.1`), and **there is no fixed `urllib3` inside the
Cashfree constraint** — every advisory's fix is above 2.1.0, and dropping to `1.26.20`
clears one of seven while moving onto the legacy line. `requirements.txt` is unchanged.
Do not spend more time on combinations; there are none.

---

## Final QA ride

**GREEN. 30 of 30**, trip 51, against QA at `a640e1b`.

Rider auth, driver auth, fare quote 135.22, driver online, ride request, dispatch
(1 driver notified), driver offer, accept, rider assignment, reached, OTP, start, 36 GPS
frames with all 36 relayed to the rider, SOS recorded and deduplicated, complete,
correlated command ack, cash confirm, fare snapshot, `final_fare` None, driver earnings,
wallet, rider history, then the idempotency retries: retried complete `already_done`,
retried cash confirm committed, receipt resend accepted, economics unchanged, no duplicate
wallet credit.

One stage passes while proving less than its name suggests, and the harness says so in its
own output rather than in a footnote: `OPS_VISIBILITY` returns 401, which shows the
endpoint exists and is protected, annotated *"NO QA OPERATOR ACCOUNT EXISTS so
authenticated operator visibility is UNPROVEN."*

The ride failed on its first attempt, and the reason is the blocker itself — the abandoned
trip 49 from the Branch B drill was still holding its driver out of supply because no
operator existed to resolve it. Resolved through the product, by the driver returning and
finishing their own ride. No shell, no SQL, `final_fare` still None.

---

## Final security negative test

**GREEN.** Repeated after every auth change in this run. The matrix:

| Caller | Six privileged endpoints | Payout approval | Own KYC |
|---|---|---|---|
| anonymous | 401/403 | 401/403 | — |
| rider token | 401/403 | 401/403 | — |
| driver token | 401/403 | — | refused, not approved |
| expired token | 401/403 | — | — |
| malformed token (incl. `alg:none` claiming admin) | 401/403 | — | — |
| forged but validly signed `role`/`is_staff` claim | 401/403 | — | — |
| self-registered account | 401/403 | 401/403 | — |
| `role=admin` without `is_staff` | 401/403 | 401/403 | — |
| rider carrying `is_staff` | 401/403 | 401/403 | — |
| support-scoped operator | 200 (read) | **401/403** | 401/403 |
| finance-scoped operator | 200 (read) | allowed | 401/403 |
| legitimate operator (admin) | **200** | **allowed** | **allowed** |
| superuser | **200** | allowed | allowed |

The malicious self-admin attempt still fails, verified against the running QA deployment
and not only locally.

---

## Remaining engineering blockers

Everything else on this list is human or external.

1. **FCM token rotation** — no `onTokenRefresh` listener; a long-signed-in driver can hold
   a token the server does not have. Needs a mobile change and a refresh endpoint.
2. **Celery beat topology** — still `-B` inside the worker in the deployment. Correct for
   one worker, doubles every scheduled task at two. The target shape is already exercised
   in the load stack.
3. **Observability** — `SENTRY_DSN` unset everywhere. A production exception would be
   visible only in container logs.
4. **The Next.js console** — now able to sign in for the first time (it declared
   `role: 'admin'`, which the backend correctly refuses, and read `data.access` where the
   backend returns `data.token`, so it never stored a token and looped back to `/login`).
   Still not pilot-ready; the Django console remains canonical.
5. **Razorpay residual** — three `RAZORPAY_*` variables are still read in settings
   although only Cashfree remains. Dead configuration; delete the reads so nobody later
   assumes a second gateway is wired.

---

## Remaining human / external blockers

1. **Rotate the database credential**, or confirm it is already dead. Holds the gate RED
   on its own.
2. **Run `bootstrap_qa_operator`** against QA. One command; unblocks four scorecard items
   and four of the nine questions below.
3. **Move the production deploy trigger off `dev`**, and resolve the staged delete, before
   configuring production.
4. **Restrict the Maps keys** — three keys, quota, alert.
5. **Watch a real handset receive a backgrounded ride offer.**
6. **Rehearse a database restore** before a real rider exists.
7. **Cashfree sandbox credentials**, for the provider-contract plan.
8. **Provision the production environment** (depends on 1 and 3).
9. **S3 bucket policy** — private, per-environment, least privilege.
10. **India legal and business sign-off** — out of scope for this run, and unchanged.

---

## Final answers

**Can we onboard 10 drivers without engineering intervention? — NOT YET.**
The driver's half is proven end to end. The operator's half — find, review, approve — has
never been performed, because no operator account exists.
*Minimum remaining condition: run `bootstrap_qa_operator` against QA, then approve one
real driver through the console.*

**Can an operator run the pilot without developer tools? — NOT YET.**
Every capability an operator needs is built, authorized correctly, audited and tested.
Nobody has ever signed in.
*Minimum remaining condition: the same one command, then one rehearsal pass over the
workflows — KYC, stale ride, SOS, support, payout.*

**Can a backgrounded driver receive an offer? — NOT YET.**
Server path tested, client handler present and degrading quietly, no device observed.
*Minimum remaining condition: one physical Android handset with a real Firebase project,
app backgrounded, screen off, one dispatch sent.*

**Can we run 100 controlled cash rides? — YES.**
100 of 100 completed on an isolated stack, with every financial invariant reconciled
afterwards: one settlement each, commission exact to the paisa, `final_fare` NULL, no
duplicate wallet movement, no overlapping driver assignment.
*Caveat, not a condition: that was an isolated single-machine stack. It is not a capacity
claim.*

**Can Celery recover critical work within an acceptable operational window? — YES.**
No work is lost when a worker is SIGKILLed, proven against a real broker. Every task is
classified by recovery objective, and the one task that could not live with the broker's
~17 minutes — the rider-visible accept deadline — now recovers in ~3 through a
database-backed path that does not use the broker at all.
*Residual, accepted and documented: everything else still recovers on the broker's ~17
minutes, which is appropriate for a receipt and for a metrics computation.*

**Can we safely create production infrastructure? — NOT YET, but nearly.**
The topology, release strategy and variable checklist are prepared, the boot guards are
proven by real subprocess boots, and the variable set is rehearsed.
*Minimum remaining condition: move the deploy trigger off `dev` first, and answer the
database-credential question. Provisioning before either is done creates a live
production that releases on every merge, possibly with a compromised credential.*

**Can we deploy production? — NOT YET.**
Nothing to deploy to; it builds a different artifact from QA; no successful deployment
ever; no rollback target; no rehearsed restore.
*Minimum remaining condition: the full ordered sequence in the production package —
trigger, credential, database, Redis, variables including `RAILWAY_DOCKERFILE_PATH`,
worker, beat, eviction policy, bucket, restore drill — then a tagged deploy verified
through `/version`.*

**Can we enable online payments? — NOT YET.**
No Cashfree sandbox proof exists, and the SDK pin is also holding 14 CVEs in place.
*Minimum remaining condition: sandbox credentials, then the eight-step provider-contract
validation. The SDK upgrade passing 860 tests proves nothing about provider behaviour.*

**Can we enable automatic payouts? — NOT YET.**
Same blocker, higher stakes: a payout moves money out of the platform.
*Minimum remaining condition: the same sandbox proof, plus a demonstrated duplicate
`transferId` rejection and a demonstrated timeout becoming `unresolved` rather than
`FAILED`. Until then, payouts stay manual and an unresolved payout stays unresolved
rather than being retried or refunded.*

---

## Engineering — next 5

Risk ordered.

1. **Close the FCM token-rotation gap.** An `onTokenRefresh` listener in the driver app
   and a narrow endpoint for it to call. *It is the same silent-push failure just fixed,
   arriving by a different route, and it fails identically: nothing raised, nothing an
   operator can see, a driver who stops getting background offers.*
2. **Move beat out of the worker.** The load stack already runs it as its own service, so
   this is a deployment change with the topology already exercised. *It is survivable at
   one worker and doubles every scheduled task the moment there are two — including the
   money sweeps and the new accept-deadline sweep.*
3. **Wire Sentry.** `SENTRY_DSN` in QA first, then production. *A pilot with no error
   aggregation finds out about exceptions from drivers, and the whole point of the
   operator console is to not learn things that way.*
4. **Delete the Razorpay reads and finish the Next console, or retire it.** *Dead
   configuration that looks live is how someone later assumes a second gateway exists;
   and a console that can now sign in but is not pilot-ready is an ambiguity in the middle
   of the operations story.*
5. **Extend the load harness to the rider-app path that misses the OTP push.** *The
   backend fix is measured and the app fix is tested in isolation, but nothing yet proves
   the two together under the concurrency that produced the miss.*

## Human / external — next 5

Risk ordered.

1. **Rotate the database credential, or confirm it is dead.** *It holds the gate RED on
   its own, independently of every application item in this report, and no amount of
   engineering closes it.*
2. **Run `bootstrap_qa_operator` against QA** and hand over the credential.
   ```
   railway run --service backend --environment QA \
       python manage.py bootstrap_qa_operator
   ```
   *One command. It unblocks four scorecard items and four of the nine questions above —
   more than anything else on either list.*
3. **Move the production deploy trigger off `dev`, and resolve the staged delete.** *Today
   those builds fail harmlessly. The moment production has variables and a domain, every
   merge to `dev` ships — a configuration that produces accidents rather than decisions.*
4. **Restrict the Maps keys and set a billing quota.** *An unrestricted key in a shipped
   APK is extractable by anyone who downloads it, and the bill is the platform's.*
5. **Watch a real handset receive a backgrounded ride offer.** *Push is the fallback
   dispatch channel and whether it works has never been observed. Nothing in a test suite
   can substitute for it, which is exactly why that item is YELLOW.*

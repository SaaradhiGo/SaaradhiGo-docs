# India launch closure report

Written 2026-09-24 after an extended autonomous engineering run.

Every claim here is traceable to a test run, a tool response, or a commit. Where
something is unverified, it says so. Test counts appear as evidence of specific
behaviour, never as a summary of readiness.

---

## 1. What changed, and how it was verified

### Priority 0 — the platform was shipping two brand names

Raised first and treated first, because it was a real customer-visible defect
rather than cosmetics.

A rider received an SMS saying "Your OTP for VahanGo", opened an app labelled
SaaradhiGo, found a wallet the **backend** told it to call "VahanGo Credits", and
on completing a ride got a GST receipt headed "VahanGo / SaaradhiGo Mobility"
thanking them for riding with VahanGo. Driver grievance and fraud addresses
pointed at `@vahango.com`, a domain this platform does not use — a compliance
problem, not a typo, since a grievance address nobody reads is worse than none.

Customer-facing copy now reads `PLATFORM_BRAND_NAME`, `PLATFORM_LEGAL_ENTITY` and
`PLATFORM_CONTACT_DOMAIN`. Brand and legal entity are kept as separate facts.

**Verified on the deployed QA stack**, not just locally:

```
[08:12:45] REMOTE_CONFIG http=200 wallet={... "display_name": "SaaradhiGo Credits"}
[08:12:45] CHECK wallet_label_is_saaradhigo   result=PASS
[08:12:45] CHECK no_retired_brand_in_config   result=PASS
[08:12:45] SUMMARY failures=0
```

An earlier run of that same check reported FAIL. The QA deployment had not gone
live yet — `/healthz` returned 200 from the **old** container. The Railway
deployments API showed `status: QUEUED` at the moment of the probe and `SUCCESS`
three minutes later. Health endpoints do not prove which code is running.

9 AST-based backend guard tests plus 3 Dart ones stop the old name returning,
while leaving comments and docstrings that explain the migration legal. Three
existing tests had **pinned the old brand** — including one asserting
`b'VahanGo' in pdf or b'SaaradhiGo' in pdf`, an "either brand" assertion that is
precisely how an inconsistency survives a test suite.

Left alone deliberately: the store bundle identifiers (`com.deeptrics.saaradhigo`
vs `com.saaradhigo.driver` — who owns the listings is a business decision) and the
legal entity name on the MVA-2020 receipt statement, which needs an accountant,
not an engineer.

### Priority 1 — a Flutter toolchain now exists

```
Flutter 3.47.5 · channel stable · revision 6a19cca564
Tools · Dart 3.13.4 · DevTools 2.60.0
```

Satisfies both apps (rider needs Dart `^3.11.1`, driver `^3.12.2`). This is the
precondition for everything else in the mobile half, and its absence is what let
false explanations accumulate (see §2).

`flutter test` and `flutter analyze` work. **`flutter build apk` does not** — see
§4.

### Priority 2 — the rider test quarantine was hiding four real defects

Nineteen rider tests were quarantined. **Every stated reason was wrong.** They
were inferences written when nobody could run the tests.

Seven ride-state-recovery tests were blamed on a SharedPreferences mock leak. No
leak existed. The tests were right and the app was wrong:

| Defect | What a rider experienced |
|---|---|
| Both recovery paths dropped the trip id — `syncStateFromBackend` receives it as a parameter, the other parses it into a local, neither stored it. `_saveActiveTripId()` only keeps `active_trip_id` while `state.tripId` is set, so recovering a ride **deleted the pointer it had just recovered from**. | App killed mid-ride → ride lost on the next offline start. |
| **The status vocabularies did not match.** The backend's `TripStatus` choices are `requested`/`accepted`/`reached`/`in_progress`/`completed`/`cancelled`. Both recovery ladders branched on `'arrived'` and `'started'`, which the backend never sends, and fell through to the current status otherwise — on a cold start, `RideStatus.none`. The live WebSocket handler had `'reached'` right; only recovery was wrong. | **App restarts while the driver waits at the pickup point → the rider sees no active ride at all.** Recovering mid-search failed identically. |
| `clearState()` was `void` while firing two un-awaited SharedPreferences writes. | A cancel or completion followed closely by an app kill could leave a finished trip's id on disk. |
| The cancelled-overlay auto-dismiss used bare `Future.delayed`: uncancellable, and it wrote to a disposed notifier if the rider left within three seconds. | Thrown exception; two cancellations left two timers racing. |

Also found: `RideState` was `@immutable` with **no `operator ==`**, so it compared
by identity and `NotifierProvider` rebuilt every listener on every write — once
per driver-location tick, on the screen a rider watches for an entire ride.

And a test named `'saves active trip ID when backend has active trip'` **asserted
nothing at all.** It built a fixture, noted in a comment that `RideService` could
not be injected, and ended — reporting green while covering nothing.

Two test-side corrections, both corrections to tests rather than to production:
one test asserted `const RideState()` **and** `showCancelledOverlay == true` on
adjacent lines, which cannot both hold; and fixtures used `'started'`,
`'ride_started'` and `'driver_accepted'`, none of which the backend sends
(`'ride_started'` is a WebSocket *event type*, not a status — that conflation is
what hid the `reached` bug).

Twelve screen tests were blamed on "a redesign whose widget tree differs from what
ships. Needs a device." They are widget tests; they never needed a device. They
were hitting `ProviderNotFoundException` — `HomeScreen.initState` reads
`MapProvider` and `NotificationProvider`, and each suite registered only
`AuthProvider`. Five now pass behind a shared harness. Three app-side defects
surfaced: `privacy_security_screen.dart` passed each widget's own `key` down onto
a descendant at three sites, so the same key appeared **twice in the tree** and
every `find.byKey()` on it was ambiguous — unusable for tests, integration
drivers and accessibility tooling alike; and two screens were missing test hooks
entirely.

**Rider tests: 31 passing → 67 passing.** Quarantined bodies: 19 → 8, each of the
8 now carrying an accurate reason instead of one false suite-level claim.
`SaaradhiGo-docs/runbooks/rider-test-debt.md` classifies all 19.

### Priority 4 — the rider can now see how the fare was calculated

The booking screen showed one number per vehicle type and no way to ask why, while
`/ride/estimate-fare/` was already returning `base_fare`, `distance_fare`,
`time_fare`, `surge_multiplier` and `min_fare_applied`. A finished transparency
panel existed and had never been rendered, behind an `ignore_for_file:
unused_element` and a note saying it "needs to be seen on a device". It needed a
toolchain.

It is now `lib/widgets/fare_breakdown.dart`, reachable from the booking screen.
The panel does no arithmetic — it renders the server's numbers, because the
backend's own comment records that trusting client-supplied distance was the
audit's primary fare-tampering vector.

Three things the app now says that it never did:

- **Surge is disclosed.** A multiplier above 1x is named and explained. India's
  MVA-2020 guidelines both cap surge and require disclosure, and the rider was
  previously paying one with no indication at all.
- A minimum fare that kicked in is stated — exactly where a rider's arithmetic
  stops matching ours.
- The total is labelled an estimate that can change.

Two defects fixed in the widget while moving it: it compared
`fareBreakdown['discount'] > 0` on a dynamic, and DRF sends `"0.00"` as a String,
so that throws at runtime; and an inactive promo must not render a "Promo
discount" line worth nothing. 17 tests, including one asserting that
`FarePricing`, `RateCard`, `fare_basis` and the raw field names never reach a
rider.

### Priorities 12–14 — the duplicate operations console, decided

Two live consoles, neither declared canonical. **Decision: KEEP the Django
console (`servers/admin_dashboard/`) as canonical for the pilot; migrate later,
deliberately.** Full reasoning in
`SaaradhiGo-docs/runbooks/operations-console-decision.md`.

Decided on evidence. The Next.js console's **driver-approval page was broken** —
it called `/driver/admin/list/`, which does not exist. Probed unauthenticated
against deployed QA, where 404 means no such route and 401 means exists-and-
protected:

```
/api/v1/driver/admin/        401  EXISTS
/api/v1/driver/admin/list/   404  MISSING
```

The page rendered "Could not load drivers". That is the screen an operator uses to
approve the first ten drivers. All 19 of its endpoints were probed; the other 18
exist and the SOS action names match the backend exactly. So it was **one typo
from correctly wired** — and nothing in the repository could have said so, because
there was no CI and no tests.

Fixed, and CI added: `.github/workflows/ops-console.yml` runs lint, type-check and
build. **The first CI run in that repository's history passed.** Two traps found
while adding it: there was no ESLint config at all, so `next lint` *prompted*
interactively — which hangs a runner rather than failing it — and `next lint` is
deprecated in Next 15 regardless.

The Django console is a genuine superset (19 pages vs 11), its auth was read and
verified rather than assumed (`admin_required` requires authenticated **and**
admin-or-superuser, applied to every view), and it has test coverage.

### Priority 31 — the test isolation defect, root-caused

CI ran the PostgreSQL-marked tests **one process per file across ten files**, with
a comment admitting nobody knew why a single process failed eight of them, and
inviting whoever root-caused it to collapse the job back.

In a single process it was **22 failures out of 97** — including *every*
trip-request idempotency test and most of the reconnect state machine. The tests
that prove this platform's riskiest invariants were the ones that could not be run
together.

**Root cause, and it was not what the comment guessed.** The Channels channel
layer is a process-wide singleton, and a `RedisChannelLayer` binds its connections
and pending futures to whichever event loop first touched it. Every async test
gets a fresh loop, so the second async test inherited a layer wired to the first
test's dead loop:

```
RuntimeError: Two event loops are trying to receive() on one channel layer at once!
```

Consumer tasks outliving their tests made it worse —
`LocationBroadcastMixin._location_pump` sitting in `await queue.get()` keeps a
`receive()` outstanding against the shared layer.

A root `conftest.py` now clears that cache around every test. **22 failures → 1**,
and the run got 110 seconds faster. CI now runs **two processes instead of ten**,
and `postgres-concurrency` **passed in CI** on that configuration — 27 + 70 = all
97 postgres tests.

### The defect that noise was hiding

Chasing the last failure found a real production defect. `get_driver_active_trip`
caught every exception and returned `None`. `None` is also what it returns for an
idle driver, so **every caller read a Redis failure as availability**:

- `add_driver_location` skipped the `zrem` that removes a busy driver from the geo
  index, leaving a driver already on a trip discoverable as nearby and free.
- `find_nearby_drivers` reported `status: 'online'` instead of `'busy'`.
- The admin driver listing showed a driver on a trip as online.

Nothing logged above DEBUG. A Redis blip quietly degraded **driver exclusivity**
and showed operators the wrong state. The row lock in the trip-accept path is what
actually stops a double assignment from committing — so this was not money loss —
but a driver being offered rides it cannot take, and a console that disagrees with
reality, are real failures.

It now raises `DriverTripStateUnavailable`, and each of the three call sites picks
an explicit conservative default: busy for dispatch, skipped in a nearby search,
`'unknown'` for operators rather than a confident `'online'`. `set_driver_active_trip`
returns the actual result of `set()` instead of `True` unconditionally. 9 tests.

---

## 2. The pattern worth naming

Four separate times tonight, a **stated explanation was wrong**, and each one had
stopped someone from looking:

| Stated | Actual |
|---|---|
| "SharedPreferences mock state leaks between tests" | The app dropped the trip id and misread the backend's status vocabulary. |
| "Needs a device to see the real layout" | Missing providers in the test harness. No device required. |
| "A SET reports success and an immediate GET returns nothing, with no client error raised" | There was a client error. The code was swallowing it — and that swallow was itself a production defect. |
| "Not wired because it needs to be seen on a device" (fare panel) | It needed a Flutter toolchain. |

Every one of these was written in good faith and sounded plausible. None had been
tested. The common cause is that **nobody could execute the code**, so hypotheses
became documentation. The toolchain was not a convenience; it was the thing
blocking four investigations at once.

A quarantine reason that is a guess is worse than no reason, because it is
load-bearing.

---

## 3. Verified defects not yet fixed

| # | Defect | Evidence | Severity |
|---|---|---|---|
| 1 | **No working production environment.** Crash-loops on `ALLOWED_HOSTS must be set`. `replicaStatus: crashed 1, running 0`. | Deploy logs of production's last successful build, `795b8186` | Launch-blocking |
| 2 | **Production has zero application variables.** Only Railway's injected `RAILWAY_*`. No DB, Redis, signing keys or payment config. | Railway `list-variables` on the production service | Launch-blocking |
| 3 | **Production deploys from the `dev` branch.** Failed deployment `b5450d1a` carries `commitHash 670cf5a`, pushed during ordinary development. No release branch, no staging gate, no human step. | Railway deployment metadata | Launch-blocking |
| 4 | **A database password is in git history.** `POSTGRES_PASSWORD`, commit `9af854c`, `docker-compose.override.yml:34`, 2026-05-07, reachable from `dev` and 5+ branches. ~4.5 months exposed. Value not reproduced anywhere. | `git show` of that path, key name only | Security incident — rotation required |
| 5 | **Secrets are not sealed.** `sealedVariableNames: []` on both QA and production, so every secret is an ordinary readable variable. | Railway `list-variables` | High |
| 6 | **The GitHub Actions deploy job has never worked.** `EC2_HOST_KEY` does not exist, and `ssh-keyscan` produced nothing, so the EC2 host did not answer on port 22 from the runner. | `gh secret list` (names only) + job logs | High |
| 7 | **The pipeline has been red on every push** while `test`, `lint` and `postgres-concurrency` were all green — only `deploy` fails. An always-red pipeline hides a real failure. | Runs `35968524264`, `35972510640`, `35981998748` | High |
| 8 | **The admin login has no rate limiting and no MFA**, at the backend root, in front of payout approval, refunds and KYC. | Read of `servers/admin_dashboard/views.py` | Launch-blocking for money |
| 9 | The "location permissions are permanently denied" SnackBar renders **over the bottom navigation bar** and swallows taps on it for its full duration. Denying location is common in India. | Observed directly in a widget-test probe | Medium |
| 10 | The ops console's Drivers page reads **page 1 only**; the backend paginates at 10. Fine for ten pilot drivers, wrong for a fleet. | Code + endpoint behaviour | Low for pilot |
| 11 | **Possibly real, unconfirmed:** a profile save does not read back the phone number (`Expected '9876543210', Actual null`). | Skipped test with documented reason | Unknown — must be answered |
| 12 | **Possibly real, unconfirmed:** a privacy toggle does not survive a provider rebuild. | Skipped test with documented reason | Unknown — must be answered |

Items 11 and 12 are marked unconfirmed because I did not confirm them. **Do not
read the passing test count as evidence that profile saves or privacy toggles
work.**

---

## 4. Reproducible blockers

**Windows symlink support blocks APK builds.**

```
Building with plugins requires symlink support.
Please enable Developer Mode in your system settings.
```

Deterministic, and it stops *after* dependency resolution succeeds — so
`flutter test` and `flutter analyze` still work. Remediation: enable Developer
Mode (`start ms-settings:developers`), or set
`HKLM\...\AppModelUnlock\AllowDevelopmentWithoutDevLicense=1` (needs elevation;
this session was not elevated, and a machine-wide Windows security setting should
not be changed silently), or build in CI where the runner is Linux.

Consequence to state plainly: **no APK was produced or installed from this
machine, and nothing in either mobile app has been verified on a device or
emulator.**

**One test isolation failure remains.**
`test_lifecycle_under_gps_load.py::test_the_driver_does_not_receive_its_own_location_echo`
fails when that file runs after `test_driver_active_trip_contention.py` —
reproducible in ~100 seconds with just those two files, which is a large
improvement on "only in the full suite". Its active-trip key is deleted between a
**confirmed** SET and an immediate GET in the same thread, so something outside
the test clears it. The file passes alone (27 passed). Narrowed, not closed. CI
runs that one file in its own process.

---

## 5. Infrastructure and release artifacts produced

- `.github/workflows/ops-console.yml` — first CI in the web repository; lint,
  type-check, build; green on its first run.
- `apps/web/.eslintrc.json` and a non-interactive lint script.
- Backend `postgres-concurrency` collapsed from ten processes to two; verified
  green in CI.
- Root `conftest.py` isolating the channel layer.
- `test/support/screen_test_harness.dart` for the rider app.
- `.gitignore` and `ruff.toml` now cover any local virtualenv — a bare
  `ruff check .` in a checkout with a `.venv-local` reported 7859 errors from
  site-packages, which is how a clean lint gate stops being trusted.
- Runbooks: `rider-test-debt.md`, `operations-console-decision.md`,
  `production-environment-gap.md`, and this report.

## 6. Current gate status

| Repository | Gates |
|---|---|
| `SaaradhiGo-backend` `dev @ 670cf5a` | 644 passed, 9 skipped (default suite); all 97 postgres tests pass in 2 processes; ruff clean; no pending migrations. CI: `test` ✓ `lint` ✓ `postgres-concurrency` ✓ `deploy` ✗ (pre-existing, §3 #6) |
| `SaaradhiGo-mobile` `dev @ 31ddf31` | 67 passed, 8 skipped; analyze: no errors, no warnings, 29 pre-existing infos. CI green. |
| `SaaradhiGo-web` `develop @ a798982` | lint ✓ type-check ✓ build ✓ (11 routes). CI green. |
| `SaaradhiGo-driver` | Untouched this pass. |

## 7. What this run did not cover

Of the 80 priorities, a minority were reached. Named honestly, because a gap you
know about is cheaper than one you assume away:

- **Driver app**: not touched at all. No UI audit, no command-ACK surfacing, no
  token-expiry or offline behaviour work.
- **Money**: no payout ambiguity containment, no Cashfree adapter boundary, no
  reconciliation work, no receipt reliability testing beyond the brand change.
- **Celery and Redis**: no task inventory, no worker-loss tests, no Beat
  separation, no Redis failure or stream-recovery testing. The Redis defect in §1
  was found incidentally, not by that review.
- **Safety**: no SOS operational review beyond confirming the console's endpoints
  exist. SOS reliability tests exist and pass, but were not extended.
- **Security**: no dependency audit, no WebSocket rate-limit or abuse review, no
  log-privacy review, no KYC document security review.
- **Data**: no GPS retention or privacy review, no index audit, no timezone or
  money-precision audit, no phone-normalisation or OTP-abuse review.
- **Operations**: no stuck-ride detection, no ops dashboard metrics, no analytics
  events, no load baseline, no incident drills, no QA data reset, no pilot city
  configuration or seeding.
- **Release**: no backup verification, no restore rehearsal, no rollback
  procedure, no app-store readiness check, no India compliance review.
- **Accessibility, error messages, loading states, maps, notifications**: not
  reviewed.

---

# IF WE LAUNCHED IN INDIA TOMORROW

Launching tomorrow is not possible. Not because of product quality, but because
**there is nothing to launch onto**: the production environment crash-loops on a
missing `ALLOWED_HOSTS`, holds zero application variables, and deploys from a
development branch. The only running deployment is QA.

If that were fixed overnight and a pilot ran anyway:

### A rider

Would get a working, coherent app with one brand name and, for the first time, an
itemised fare that discloses surge. The recovery defects fixed tonight mattered
most to them — before today, a rider whose app restarted while the driver waited
at the pickup point saw **no active ride at all**.

What could still go wrong: they deny location permission — common in India — and a
SnackBar covers the bottom navigation, swallowing their taps. Nothing in either
mobile app has been verified on a real device. Their profile save may not persist
their phone number (unconfirmed, §3 #11).

### A driver

Their app was **not touched tonight**. Their grievance and fraud contact addresses
now point at a domain this platform actually uses, but **nobody has confirmed
those mailboxes exist and are monitored** — which is the part that matters for
compliance. KYC document security was not reviewed. No APK has been built or
installed, so the driver app's behaviour on a real handset is unverified by this
pass.

### An operator

Now has a stated canonical console, which is an improvement on two consoles and no
default. The driver-approval page works rather than showing "Could not load
drivers". When Redis cannot answer, they will see `unknown` instead of a confident
`online` that is really a failure.

What could still go wrong: the admin login has **no rate limiting and no MFA**, at
the backend root, in front of payout approval and refunds. There is no stuck-ride
detection. There are no operational dashboards or alerts from this pass, so the
first sign of trouble will be a phone call.

### The business

The exposed database password is the sharpest risk: **live for four and a half
months, in every clone of the repository, and not yet rotated.** Until it is
rotated, the database's security rests on network reach alone.

Financially, the money chain's invariants are backed by tests that now actually
run together — that is a genuine improvement, since the trip-idempotency tests
could not previously be run in the same process as their neighbours. But no payout
or reconciliation work was done tonight, and the always-red CI pipeline is a
cultural risk: a team that has learned to ignore a red pipeline will ignore the
run that matters.

---

## What prevents SaaradhiGo from safely onboarding the first 10 real drivers and serving the first 100 real rides?

Six things. In order.

**1. There is no production environment.** It crash-loops on `ALLOWED_HOSTS` and
has zero application variables. This is configuration, not code — the boot guard
is working correctly by refusing to serve — but until it is done there is nowhere
for 10 drivers or 100 rides to exist.

**2. Production deploys from `dev`.** Configuring production while its service
still tracks a development branch converts a dormant misconfiguration into a live
one. Repoint it **before** populating it, or the first push after configuration
deploys whatever a developer happened to commit.

**3. A database password has been public for four and a half months and has not
been rotated.** Rotation is the only action that ends this. Rewriting git history
first would leave a live credential in every existing clone while creating the
impression the problem was handled.

**4. The console that approves drivers and releases money has no rate limiting and
no MFA.** Ten drivers means ten KYC approvals and ten payout relationships,
performed through a login protected by a phone number and a password at a public
root.

**5. Nothing has been verified on a device.** Both mobile apps are tested only in
`flutter test`. The APK build is blocked by a Windows symlink setting, so no build
has been installed and run. Onboarding 10 real drivers means 10 real handsets, and
this pass produced no evidence about any of them.

**6. Two possible defects in the rider's account surfaces are unresolved** — a
profile save that may not persist a phone number, and a privacy toggle that may not
survive a rebuild. Both are cheap to settle and neither should be settled by
assuming the test was wrong, which is exactly the mistake this run spent the night
undoing.

Nothing on that list is a deep architectural problem. Five of the six are
configuration, credentials, or verification that has not happened yet. That is a
better position than a design flaw — but it is not readiness, and the number of
passing tests does not change it.

One further caution. Of the 80 priorities set for this run, a minority were
reached (§7). The driver app, Celery, Redis failure behaviour, payouts, SOS
operations, dependency security and data retention were **not reviewed**. The
absence of findings in those areas is an absence of looking, not a clean bill of
health.

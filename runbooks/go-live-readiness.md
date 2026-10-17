# SaaradhiGo — India controlled-pilot readiness

> ## Status: autonomous batch 6, 2026-10-17
>
> Read `android-pilot-autonomous-batch6-report.md` for the detail. In short:
>
> **Nine defects found and fixed**, each with a test that fails against the
> previous code. Two were security findings not previously reported: the
> Cashfree `payment_session_id` was written to logcat on every payment-order
> creation, and the driver's push token was logged once the app began
> registering rotated tokens. Both are fixed, in both apps, by substring
> redaction with a structural test.
>
> One was a P0 functional defect: **a rider could not cancel a ride.** The app
> sent no `reason`, the server refused with `INVALID_REASON`, and the app
> cleared its local state anyway -- so the rider was shown "cancelled" while
> the trip stayed live and a driver could still be dispatched to them.
>
> **Three rider capabilities now exist** that did not: support tickets (the
> Help & Support screen had six tiles and no handlers), post-trip ratings
> (neither app could submit one, so every driver average sat at its default),
> and a notification error state distinct from an empty inbox. The driver can
> now rate passengers.
>
> **The ride lifecycle is pinned to a generated contract**, 17 envelopes
> captured from real views and asserted in both apps -- the same technique
> that caught the `access`/`token` login defect, extended past authentication.
> **28 object-authorization probes** confirm the boundaries hold.
>
> **CI certified 2026-10-17.** All four branches pushed; **9 of 9 jobs green**
> across driver, rider and backend. Backend CI: **1,417 passed**, and the 9
> tests that fail locally for want of Redis pass there, as do the **102
> PostgreSQL concurrency tests** that no local run can execute -- both were
> UNKNOWN until this push and are now PASS. Driver **459** tests, rider
> **259**; both `release-config` jobs confirm the signing guard still refuses
> an unsigned release build with no secrets on the runner.
>
> Two further defects were found during the CI/release recheck and fixed:
> **the driver app's REST base URL had no `/api/v1`**, so every REST call in
> it 404d -- login included -- and the build script had given the driver the
> rider's opposite WebSocket convention, which would have produced
> `/wsws/...`. Both apps now pin URL composition against real backend routes.
> And **rider cancellation now requires server confirmation before any local
> state is cleared**, with a failure preserving recoverable trip state and
> telling the rider.
>
> The four owner-controlled blockers are unchanged: keystore backup
> verification, Firebase credentials, a QA Maps key, and QA SES. Every push,
> Maps and email row in the device checklist fails for want of one of them.



Written for: the owner/CTO and the engineers who will clear these items.

**Audited at:** backend `4228caf`, rider `69598ae`, driver `bae0a3e`, web `fcf56fc`,
docs `5fff7bd`. All five working trees clean. Audit only — nothing was fixed,
merged, provisioned or deployed; no Railway configuration was changed.

> **UPDATED 2026-10-17.** Two batches of work have since landed on
> `feat/android-pilot-batch1` in three repos (nothing merged to `dev`). See
> `android-pilot-batch1-report.md` and `android-pilot-batch2-report.md`. Rows
> below that this work changed are marked **[FIXED]** or **[CORRECTED]**; the
> audit's own errors are corrected in place rather than left to mislead.

**Verification policy:** anything not actually executed in this session is marked
**UNKNOWN**, never PASS. Environment limits are listed in §7.

---

## 0. The headline

The backend is the strongest part of the system and is not the blocker. The
blockers are in the **driver app** (a driver cannot submit documents or add a
vehicle from the app, and release builds are signed with the debug key), one
**live QA auth defect found during this audit** (nobody can log in to QA right
now), and **one regression this session's own backend work introduced into the
rider app's fare screen**.

Rider app **≈72%**, driver app **≈78%** of launch-critical journeys (method in
§2/§3). Neither number is a code-completeness figure — both are journey counts.

---

## 1. Readiness table

| Area | Implemented | Missing | Priority | Evidence |
|---|---|---|---|---|
| Driver document upload / identity verification **[CORRECTED]** | Backend accepts documents; console reviews them. **`origin/dev`'s driver app already has a full presigned-upload implementation** (`core/uploads/presigned_upload_service.dart`, a document vault and onboarding screens) — the audit checked only `feat/driver-app-rebuild`, the branch the repo was on, and reported a branch-specific gap as a product-wide one. Built on the rebuild branch in batch 1 | **On the rebuild branch:** no UI anywhere. Driver app profile is read-only ("Not uploaded"); no image/file picker dependency at all; admin console has no multipart form and no `request.FILES` handling | **P0** | `SaaradhiGo-driver/lib/features/profile/presentation/profile_screen.dart:240`; `pubspec.yaml` deps = `dio` only; `grep -rn "FILES\|upload" servers/admin_dashboard/views.py` → 0 handlers |
| Driver app vehicle registration | `ApiEndpoints.vehicleAdd` declared; backend route live | No screen calls it — only `ApiEndpoints.vehicles` (list) is used | **P0** | `lib/core/network/api_endpoints.dart:30`; `lib/features/profile/data/profile_repository_impl.dart:19` |
| Driver Android release signing | Rider app reads `keystore.properties` when present | Driver `release` block hardcodes `signingConfig = signingConfigs.getByName("debug")`; rider falls back to debug when no keystore | **P0** | `SaaradhiGo-driver/android/app/build.gradle.kts:51`; `SaaradhiGo-mobile/android/app/build.gradle:78` |
| QA login (OTP) | Test-phone bypass fires correctly and sends **no** SMS | OTP is cached then reported "not found" 6s later; **no one can log in to QA**, which also disables `qa/acceptance_ride.py`, `abandoned_ride_drill.py`, `driver_self_recovery.py` | **P0** | Live: `otp_request=200`; Railway log 00:03:22 `OTP test-phone bypass … no SMS sent` then 00:03:28 `OTP expired or not found`. Redis healthy (12 MB / 24 GB). Cause **UNKNOWN** |
| Rider fare screen currency | Backend now returns `currency: "INR"` (added this session) | App does `rideData?['currency'] ?? '₹'` and interpolates it as a **symbol** → renders `INR190.00`. A regression caused by the backend addition | **P0** | `SaaradhiGo-mobile/lib/screens/home/rider_flow_screens.dart:1244,1292` |
| Rider safety / SOS | Backend `/sos/` live; driver app has a full safety screen | **CORRECTED 2026-10-17 — worse than first recorded.** The rider app had an SOS button that showed "Emergency Alert Sent! Support is on the way." and called **no API**. Not missing: falsely claiming success. No trip sharing. *(Fixed in `feat/android-pilot-batch1`.)* | **P0** | `rider_flow_screens.dart` pre-fix: `showDialog` → snackbar, no HTTP call. The original row said "no SOS" because a `grep -i` was truncated by `head`, hiding it behind ten `emergencyContact` lines — a truncated grep cannot prove absence |
| Android 14+ background location (driver) | Permissions declared incl. `ACCESS_BACKGROUND_LOCATION`; `flutter_background_service ^5.0.10` | No `<service>` element and **no `foregroundServiceType`** in the driver manifest; the rider app declares `android:foregroundServiceType="location"`. Crash risk on API 34+ | **P0 (UNKNOWN until a device run)** | `grep "<service"` driver manifest → none; `SaaradhiGo-mobile/.../AndroidManifest.xml:49` has it |
| Two-sided ratings | Backend `rate_trip` live | **Neither app submits a rating** | P1 | No `rate/` path in either app's source |
| In-app support | Backend `/support/tickets/*`; driver app has `support_screen` | Rider `help_support_screen.dart` is static — no API, no `tel:`, no `mailto:` | P1 | `grep -nE "launchUrl|tel:|api|Ticket"` → 0 matches |
| Rider receipts | Backend issues PDF + email; `/ride/trip/<id>/receipt/pdf/` live | Rider app never calls it; fare breakdown only | P1 | `grep -rn "receipt" SaaradhiGo-mobile/lib` → icon + comment only |
| DPDP data rights | `/auth/me/export/`, `/auth/me/delete/` implemented | Rider `privacy_security_screen.dart` calls neither; Play Store expects an in-app deletion path. **No retention/purge job exists** | P1 | `servers/auth_user/urls.py:25-26`; no `retention|purge` in any `tasks.py` |
| iOS (driver) | — | `ios/Runner/Info.plist` is the stock template: no `NSLocation*UsageDescription`, no `UIBackgroundModes`. The driver app cannot request location on iOS | P1 (out of pilot scope if Android-only) | plist key dump |
| iOS (rider) | Location + photo-library usage strings present | `CODE_SIGN_STYLE = Automatic`, no `DEVELOPMENT_TEAM`; no signing/distribution set up; **no Mac in this environment** | P1 | `ios/Runner.xcodeproj/project.pbxproj:387` |
| Error monitoring | `sentry-sdk[django]==1.32.0` wired behind `SENTRY_DSN` | `SENTRY_DSN` is **not** among QA's variables → Sentry is off. No alerting rules | P1 | `requirements.txt:59`, `settings.py:893`; QA variable list |
| Multi-country backend + admin | Complete and verified this session | Mobile apps unchanged: 20 hardcoded `₹` in rider, 6 in driver | P2 for the India pilot | `multi-country-admin-backend-readiness.md`; grep counts |
| Ride lifecycle, pricing, dispatch, payouts (backend) | 1,189 tests passing; drill 34/34; 7 migrations applied to QA Postgres | 9 Redis-dependent tests **UNKNOWN locally** (Docker not running); pass in CI | — | §7 |
| Railway cost control | QA runs 6 services; `/healthz`; pre-deploy `migrate` | **Every push to `dev` auto-deploys with `checkSuites: false`** — it does not wait for CI, so a red build still ships | P1 (cost + quality) | `get-service-config` for `backend` and `celery` |
| Cashfree payments | Sandbox integration, webhook secret set, currency guard added this session | Live keys not enabled (correct — outside my authority) | Owner decision | `CASHFREE_ENVIRONMENT` in QA vars |

---

## 2. Rider app — ≈72%

**Method:** 16 launch-critical journeys; Implemented = 1, Partial = 0.5,
Missing = 0. Journeys, not files or screens — a screen that exists but cannot
complete its job scores 0.

Implemented (1.0 each, 11 journeys): OTP login · profile · saved locations
(`/rider/locations/`) · location + destination search (`/ride/maps/*`) · booking
over WebSocket · driver matching · live tracking + ETA · cancellation · ride
history · wallet · network-failure recovery (`state_recovery_decision_service.dart`,
`ride_state_recovery_test.dart`).

Partial (0.5): fare estimate (currency regression) · payments (Cashfree sandbox
only) · push notifications (implemented; **background delivery unproven on a
device**).

Missing (0): in-app receipts · ratings · working support · SOS/trip-sharing
(counted inside the safety journey).

**11.5 / 16 = 72%.** Evidence: `flutter analyze` exit 0 (32 infos), `flutter test`
**77 passed / 7 skipped**.

---

## 3. Driver app — ≈78%

Same method, 16 journeys.

Implemented (1.0, 12): OTP login · KYC status visibility · online/offline duty ·
offer accept/reject · external navigation (`url_launcher`) · arrival · trip start
with OTP · completion · continuous GPS with reconnect · earnings · withdrawals ·
safety + support.

Partial (0.5): Android 14+ background location (permissions yes, service type no).

Missing (0): document upload · vehicle registration · rating submission.

**12.5 / 16 = 78%.** Evidence: `flutter analyze` → "No issues found!" exit 0;
`flutter test` **88 passed**, exit 0.

The driver app's engineering quality is high — clean feature/domain/data layering,
a manifest regression test, WS reconnect tests. The gap is one missing capability
(onboarding), not general incompleteness.

---

## 4. Architecture and integration map

```
Rider (Flutter, flutter_map/OSM)        Driver (Flutter, google_maps_flutter)
  REST /api/v1/* + WS /ws/ride/*          REST /api/v1/* + WS /driver/location/
            \                                        /
             \                                      /
            Django 5.2 + DRF + Channels (Daphne, ASGI)
            ├── PostgreSQL 15   trips, fares, payments, zones, KYC
            ├── Redis           presence/geo, dispatch, cache, Celery broker
            ├── Celery (+beat)  dispatch waves, receipts, payouts, GPS drain
            ├── AWS S3          private KYC documents (presigned PUT)
            ├── AWS SNS         OTP SMS (test-phone bypass skips it)
            ├── Firebase FCM    push to both apps
            ├── Google Maps     Distance Matrix, geocode, directions
            └── Cashfree        payments (sandbox) + UPI payouts
                     |
      ┌──────────────┴───────────────┐
  Django console (19 pages)      Next.js ops console (11 pages)
  CANONICAL for the pilot        not pilot-ready, no tests, no CI until recently
```

Two ops consoles exist; `runbooks/operations-console-decision.md` (2026-09-24)
already ruled the **Django console canonical** for the pilot. This audit does not
reopen that. Note the Next.js console has received **none** of the multi-country
work and still renders money without a currency.

**Deployment targets:** Railway QA (works, auto-deploys from `dev`) · Railway
production (`SaaradhiGo-backend` service — empty shell) · EC2 via
`.github/workflows/deploy.yml` (the only GitHub-driven deploy; targets EC2, not
Railway).

---

## 5. Backend and admin outstanding gaps

The backend has no P0 of its own. Remaining:

1. **No retention/purge job** — DPDP requires a stated retention period and
   enforcement. Export and delete exist; expiry does not. (P1)
2. **Sentry not enabled** in QA (and therefore unverified for prod). (P1)
3. **Next.js console** is money-blind and untested; keep operators off it. (P1)
4. **Production Railway service is an empty shell**, and nothing in the repos
   declares which target is authoritative for production. Owner decision. (P1)
5. `EC2_HOST_KEY` secret still unset, so the EC2 deploy job fails on a missing
   host key — a red badge on every push that trains people to ignore CI. (P1)
6. India's UPI/NPCI limits are now explicitly INR-scoped; a second payout rail
   will need its own limits. (P2)

---

## 6. Top 10 launch blockers

1. **A driver cannot be onboarded.** No document-upload or vehicle-add UI in the
   app or the console. *(P0)*
2. **Driver release APK is debug-signed** — unshippable to Play. *(P0)*
3. **QA login is broken** — OTP cached then "not found"; blocks every acceptance
   drill and all device testing against QA. *(P0, cause UNKNOWN)*
4. **Rider fare screen will render `INR190.00`** — regression from this session's
   backend field addition. One line. *(P0)*
5. **No rider SOS or trip sharing** — safety expectation for an Indian pilot;
   backend already supports it. *(P0)*
6. **Android 14+ foreground-service type missing in the driver manifest** —
   background location may crash on the QA phones. *(P0, needs a device run)*
7. **Background push to a backgrounded driver app is unproven on a device** — by
   standing rule this cannot be called ready without a real observation. *(P0)*
8. **No in-app support path for riders** and **no ratings on either side**. *(P1)*
9. **Monitoring is off** — no Sentry DSN, no alerting, so a pilot incident is
   discovered by a phone call. *(P1)*
10. **Railway deploys on every `dev` push without waiting for CI** — cost and a
    path for a failing build to reach QA. *(P1)*

---

## 7. Local testing strategy and environment limits

**Ran in this session (evidence above):** backend `pytest` 1,189 passed /
9 failed / 45 skipped; `qa/multi_country_drill.py` 34/34; `ruff` clean;
`makemigrations --check` clean; driver `flutter analyze` + 88 tests; rider
`flutter analyze` + 77 tests; one live read-only QA probe.

**Standing local-first model:**

- Backend unit/integration: `.venv-local/Scripts/python.exe -m pytest` — needs no
  Docker (`pytest.ini` → `base.settings_ci`).
- Postgres-marked tests: `postgres:15-alpine` as `sg-pg-test` on **5433**, then
  `-m postgres`.
- Isolated load: `docker-compose.load.yml` + `qa/load_harness.py` — refuses to run
  unless `/version` reports `environment: load`.
- Flutter: `$env:PATH = "C:\src\flutter\bin;$env:PATH"` then `flutter analyze
  --no-fatal-infos` and `flutter test`. **Check the exit code**, not the text.
- Mobile ride lifecycle: `SaaradhiGo-driver/integration_test/app_test.dart` exists
  and needs an emulator — **not run here**.

**Environment limitations (why things are UNKNOWN):**

- **Docker Desktop is not running** → no Redis, so 9 Redis-dependent backend tests
  (7 `test_durable_dispatch`, 2 `test_executive_revenue`) could not run. They fail
  identically on the pre-session commit and pass in CI. Dispatch timeout,
  reassignment, offer dismissal and rider-reconnect are therefore **UNKNOWN
  locally**, CI-green.
- **`flutter build apk` cannot run on this machine** ("Building with plugins
  requires symlink support. Please enable Developer Mode"). Builds go through CI;
  CI builds **debug** only. No signed-release build has been produced anywhere.
- **No Android device or emulator attached** → background location, background
  push, GPS reconnect on a real network: all **UNKNOWN**.
- **No macOS** → iOS build, signing and TestFlight: **UNKNOWN**.
- **No QA login** (blocker 3) → the authenticated live payload check and the full
  acceptance ride could not be executed against the deployed backend.

---

## 8. Railway deployment and billing control plan

**What actually triggers a Railway deploy (verified, read-only):**

| Service | Repo / branch | Waits for CI? |
|---|---|---|
| `backend` (QA) | `SaaradhiGo/SaaradhiGo-backend` @ `dev` | **No** — `checkSuites: false` |
| `celery` (QA) | same repo @ `dev` | **No** — `checkSuites: false` |

So **Railway's own GitHub watcher**, not GitHub Actions, is what deploys. Every
push to `dev` builds and deploys two services regardless of test results. The
GitHub `deploy` job (`if: push && refs/heads/dev`) targets **EC2** and is failing
on a missing `EC2_HOST_KEY`. Mobile and docs repos are not connected to Railway.

**Recommended model (requires approval — §9):**

1. Set `checkSuites: true` on both QA services so a red build cannot deploy. One
   toggle; biggest single win.
2. Move day-to-day work to short-lived branches; keep `dev` as the QA release
   branch. PR CI (lint, tests, postgres-concurrency, debug APK) runs with **zero**
   Railway spend.
3. Batch merges into `dev` — one deploy per batch, not one per commit. This
   session alone pushed 5 commits to `dev` and spent 5 QA deploys.
4. Keep the EC2 path as the production route and either fix `EC2_HOST_KEY` or
   delete the empty Railway production shell — two half-configured targets is how
   a release goes to the wrong one.
5. Test locally first (§7); reserve QA for device testing and acceptance drills.
6. Production deploy only on written sign-off.

No Railway setting was changed. Costs are also driven by Google Maps Distance
Matrix on every quote and fallback — worth a quota alarm before the pilot.

---

## 9. Items requiring owner approval

1. **Enable live Cashfree** (keys, webhook, settlement account) — nobody else can.
2. **Enable `final_fare` / metered billing** — still off by instruction.
3. **Android release keystore** for both apps, stored as CI secrets.
4. **Play Console account + listing**, data-safety declarations, privacy-policy URL.
5. **Google Maps billing + quota alarms** and a production `MAPS_API_KEY`.
6. **`checkSuites: true`** on the two QA services, and the decision between EC2
   and Railway for production.
7. **`SENTRY_DSN`** provisioning for QA and production.
8. **Credential rotation** for anything historically committed — a leaked
   credential stays compromised after removal from HEAD.
9. **Postgres backup/PITR policy** on the chosen production target — **UNKNOWN**,
   not verified in this audit.
10. **Whether the pilot is Android-only.** If yes, the iOS gaps drop to P2 and the
    plan shortens materially.
11. **Decision on the Next.js console**: freeze it, or fund it to parity.

---

## 10. Recommended first implementation batch

Scoped to one batch, one QA deploy, all P0, no new features:

**Batch 1 — "a driver can be onboarded and a rider sees a correct fare"**

1. Rider fare-screen currency: map `currency` → symbol instead of interpolating
   the code. ~1 line + a widget test. *(Fixes a regression this session caused.)*
2. Driver app: document upload + vehicle-add screens against the existing
   `/uploads/presign/` and `/driver/vehicles/add/` endpoints. The largest item,
   and the one that unblocks every real-driver test afterwards.
3. Driver `build.gradle.kts`: real release signing from `keystore.properties`,
   mirroring the rider app; fail the release build loudly when absent.
4. Driver manifest: declare the foreground service with
   `android:foregroundServiceType="location"`, and extend
   `test/android_manifest_test.dart` to assert it.
5. Diagnose the QA OTP cache miss (blocker 3). Read-only investigation first —
   it may be environmental rather than a code defect.

Deliberately **not** in batch 1: rider SOS and trip sharing (batch 2, with the
backend `/sos/` wiring), ratings, support, receipts, Sentry, retention job.

**Nothing above has been implemented. Stopping here for review.**

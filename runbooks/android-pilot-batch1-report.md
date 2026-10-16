# Android pilot — batch 1 report

Written for: the owner, reviewing before anything is merged or deployed.

**Nothing was merged, pushed, deployed, or changed on Railway.** All work is on
unpushed `feat/android-pilot-batch1` branches in three repos. Live payments and
production remain off.

| Repo | Branch | Commit | Based on |
|---|---|---|---|
| SaaradhiGo-mobile (rider) | `feat/android-pilot-batch1` | `e077dfc` | `dev` @ `69598ae` |
| SaaradhiGo-driver | `feat/android-pilot-batch1` | `d93f463` | `feat/driver-app-rebuild` @ `bae0a3e` |
| SaaradhiGo-backend | `feat/android-pilot-batch1` | `7f8c8d3` | `dev` @ `4228caf` |

---

## 1. The six items

### 1.1 QA OTP cache miss — diagnosed: **transient, not a code defect**

Non-destructive diagnosis, no OTP or secret printed.

What was observed live, from the service's own log:

```
00:03:22  INFO  OTP test-phone bypass for +9199*** — no SMS sent
00:03:22  POST /api/v1/auth/otp/  200
00:03:28  WARN  OTP expired or not found for: +9199***
```

An OTP with a 600-second TTL, gone in six seconds. What the diagnosis
established:

1. **No SMS was sent.** The bypass log line proves it, so no billing and no
   message to a real number.
2. **Redis is healthy** — 12 MB of a 24 GB limit, negligible CPU, no connection
   errors in the deploy stream. Not an eviction.
3. **The code is not the cause.** Both views derive `otp_role_<phone>` from the
   same request field with no normalisation, and `/auth/login/` does route to
   `auth_user.views.login` (checked in `urls.py`, not assumed). No
   `cache.clear()` or `delete_pattern` exists anywhere in the codebase — every
   delete is a single targeted key.
4. **It does not reproduce.** A deliberately wrong OTP now returns
   `AUTH_INVALID_OTP`, which proves the key was written *and* read back; a
   correct OTP then logged in and produced a working token.

So: **a transient fault, cause unidentified.** I did not escalate it to a
standing P0, and I did not invent a cause. Two things were added instead:

- `tests/test_otp_cache_roundtrip.py` — 10 tests fixing the contract that
  failure violated (same key both sides, survives a second request, bypass
  sends no SMS, expiry/attempts/single-use). **Stated limitation, in the file:**
  an autouse fixture in `tests/conftest.py` swaps the cache for `LocMemCache`,
  so these tests never touch Redis and would have stayed green through the QA
  failure. They stop the *code* becoming the cause; they cannot diagnose the
  environment.
- `manage.py cache_diagnose` — writes one `cache_diagnose:<uuid>` key, reads it
  back, re-reads after a delay, deletes it, and reports the backend, a
  **credential-redacted** location and the TTL the server actually applied. It
  never scans the keyspace, never calls `clear()`, and never touches a user
  key. Run it where the fault is: `railway run python manage.py cache_diagnose`.

**Consequence for the pilot:** QA login works, so `qa/acceptance_ride.py` and
the device checklist are unblocked. If it recurs, the command above is the
instrument.

### 1.2 Rider currency — fixed

The backend now returns an ISO 4217 **code**; the screen interpolated it as a
**symbol**. Confirmed against deployed QA during this batch:

```json
{"estimated_fare": "169.02", "currency": "INR",
 "fare_breakdown": {"base_fare": "60.00", "distance_fare": "55.59",
                    "time_fare": "19.62", "tax_label": "GST", ...},
 "fare_adjustments": [{"code": "surge", "amount": "33.80"},
                      {"code": "rounding", "amount": "0.01"}]}
```

60.00 + 55.59 + 19.62 + 33.80 + 0.01 = **169.02** exactly — so the decomposition
closes at the HTTP boundary, and the app would have rendered `INR169.02`.

`lib/utils/money.dart` maps code → symbol in one place, threaded through the
fare panel. A server decimal string passes through untouched rather than being
re-parsed and re-rounded; a `num` gets two decimals, because Dart renders the
double `87.5` as `"87.5"` and `₹87.5` is not a price. A pre-existing widget
test caught that, which is why the rule is in the code and in a test.

### 1.3 Driver KYC upload and vehicle registration — built

The pilot blocker. Backend endpoints existed and the console reviews documents;
no UI anywhere called them, in either app or the console.

Added: presign → PUT → submit, two screens, reachable from the profile screen
where a blocked driver is told why. The parts that are easy to get silently
wrong, and what was done about each:

| Trap | Handling |
|---|---|
| S3 rejects a Content-Type that differs from the signed one | The exact signed type is sent; asserted in a test, because Dio defaults to `application/octet-stream` |
| Leaking our Bearer token to S3 | No `Authorization` header on the PUT; the presigned URL *is* the authorisation. Asserted |
| One spinner for seven documents | Per-document state, error and progress; a retry re-uploads only the failed one |
| Submitting mid-upload | Submit disabled while any upload is in flight; otherwise the vehicle is created with whichever documents happened to finish |
| A PATCH nulling the other licence side | Only uploaded keys are sent |
| Telling a driver "Verified" | Tiles say **Uploaded**. Approval is an operator's decision; "Verified" would send a driver to try going online |
| A client-side copy of the required-document rule | Not written. Completeness is a server-side, country-driven decision |
| HEIC from an iPhone | Refused with an explanation instead of an S3 signature failure |
| An 11 MB photo on mobile data | Refused locally, before anything is presigned |

### 1.4 Android release signing — wired to CI secrets, and it now refuses

Both apps silently fell back to the **Android debug key** for release builds —
the driver app unconditionally, the rider app whenever `key.properties` was
absent, which is every CI runner. A debug-signed APK installs and runs, so this
stays invisible until Play rejects the upload.

Now: material comes from `key.properties` **or** `ANDROID_KEYSTORE_PATH` /
`ANDROID_KEY_ALIAS` / `ANDROID_KEY_PASSWORD` / `ANDROID_STORE_PASSWORD`, and a
release build with neither **stops** with an explanation. `-PallowDebugSigning=true`
is the logged escape hatch for `flutter run --release`. Debug builds are
untouched. No keystore or password is committed; `android/.gitignore` already
covers `key.properties`, `*.jks`, `*.keystore`, and a test asserts none is
present in the tree.

**CI validates this without any secret** (new `release-config` job in both
repos): an unsigned release build must fail *with the guard's message*, and the
documented opt-out must still succeed.

### 1.5 Driver Android 14+ foreground service — declared

`flutter_background_service` declares its `<service>` but **not**
`android:foregroundServiceType` (confirmed by reading the plugin's own manifest
in the pub cache). From API 34, `startForeground()` for a location service with
no declared type throws `MissingForegroundServiceTypeException` — so **going on
duty crashed the driver app on any Android 14 phone.** The rider app already
declared the override; the driver app did not.

Declared with the plugin's own service name so the manifest merger adds the
attribute rather than creating a second service, and asserted in
`test/android_manifest_test.dart` — the only check that catches this without a
device, since analyze, test and a successful APK build all pass with it missing.

### 1.6 Rider SOS and trip sharing — built, replacing a button that lied

**This was the most serious thing found in this batch, and my earlier audit got
it wrong.** The audit said the rider app had no SOS. It had one: a button that
opened a dialog claiming "This will alert the nearest authorities and our
support team", and on confirm showed

> **Emergency Alert Sent! Support is on the way.**

while calling no API at all. Nothing was persisted, no ops alert was raised,
and the emergency contact the profile screen collects was never messaged. A
rider in danger was told help was coming when nobody had been told anything.

*(The audit missed it because a `grep -i "sos\|emergency"` was truncated by
`head` — ten lines of `emergencyContact` from `auth_provider.dart` hid it. Using
a truncated grep to prove absence is the mistake; it is recorded here so the
next audit does not repeat it.)*

Now: `POST /sos/` with the category, live trip id and coordinates. **112 is
offered first**, before any of our own machinery, because the app is not a
substitute for the emergency services. A failed POST **says so** and offers the
dialler. Trip sharing opens the SMS composer with driver, vehicle, trip id and
a maps link, and invents no "live tracking" URL — no such public endpoint
exists, and a link that 404s during an emergency is worse than none.

---

## 2. Test results

### 2.1 Backend — `feat/android-pilot-batch1`

```
ruff check base/ servers/ tests/ qa/     All checks passed!
pytest                                   1199 passed, 9 failed, 45 skipped
                                         The 9 are the known Redis-dependent
                                         locals (7 durable_dispatch,
                                         2 executive_revenue) and fail
                                         identically at the branch point.
                                         Docker is not running here, so
                                         dispatch timeout, reassignment, offer
                                         dismissal and rider reconnect are
                                         UNKNOWN locally and CI-green.
tests/test_otp_cache_roundtrip.py        10 passed  (new)
tests/test_phase0_test_phone_bypass.py
  + test_auth + test_user_login
  + test_otp_delivery_retry               26 passed  (pre-existing, re-run)
manage.py cache_diagnose                  works; reports a failing SET locally
                                          with no Redis, and prints no secret
```

### 2.2 Rider — `e077dfc`

```
flutter analyze --no-fatal-infos    exit 0  (32 infos, 0 warnings/errors)
flutter test                        114 passed, 7 skipped   (was 77/7)
```
New: `money_format_test.dart` (12), `safety_sos_test.dart` (12),
`android_manifest_test.dart` (6), `release_signing_test.dart` (7).

### 2.3 Driver — `d93f463`

```
flutter analyze --no-fatal-infos    "No issues found!"  exit 0
flutter test                        124 passed           (was 88)
```
New: `kyc_upload_test.dart` (15), `kyc_screens_test.dart` (11),
`release_signing_test.dart` (7), plus 3 added to `android_manifest_test.dart`.

### 2.4 Live QA (read-only, no trip created)

- `POST /auth/otp/` → 200, bypass fired, **no SMS**
- `POST /auth/login/` → 200, token issued
- `POST /ride/estimate-fare/` → all seven market fields populated; decomposition
  closes exactly (§1.2)
- `GET /api/v1/pricing/zones/` → market fields resolved for all four zones

---

## 3. Remaining P0 blockers

| # | Blocker | Why it is still open |
|---|---|---|
| 1 | **Every device-dependent behaviour** | Cannot be cleared here: `flutter build apk` fails on this machine (symlink/Developer Mode), no emulator, no device. CI builds debug only. See `android-device-test-checklist.md` |
| 2 | **Background push to a backgrounded/killed driver app** | Standing rule: not GREEN without a real device observation |
| 3 | **Android release keystore does not exist** | Owner action. The code is ready and refuses to build unsigned |
| 4 | **Rider Maps API key is compromised** | `AIzaSy...` was committed in the initial commit and is in git history. Removed from HEAD and replaced with a placeholder — **that does not rotate it.** Rotate or restrict in Google Cloud |
| 5 | **Driver branch divergence** | `feat/driver-app-rebuild` is 19 ahead / 20 behind `origin/dev`. Must be reconciled before any release build |
| 6 | **No operator path to upload a document** | The driver app can now do it. The console still cannot, so an ops-assisted onboarding (driver emails photos) has no destination. Decide whether the pilot needs it |

Downgraded from the audit: the QA OTP failure (transient, §1.1).

Still P1, unchanged and deliberately out of this batch: ratings (the rider app
**collects** a star rating and discards it — `RateReviewDriverScreen` navigates
home without submitting, which is worse than absent), rider in-app support
(static screen), in-app receipts, Sentry/alerting, the DPDP retention job, and
in-app account deletion for Play.

---

## 4. Deliverables index

- `runbooks/railway-deploy-control-proposal.md` — the exact changes, none
  applied. **Four services auto-deploy from `SaaradhiGo-backend@dev`**, not
  two: `backend`, `celery`, an abandoned `alluring-happiness`, and the
  zero-variable production shell — the last two with deletions *staged but
  never applied*. All five services have `checkSuites: false`.
- `runbooks/android-device-test-checklist.md` — every UNKNOWN, as a checklist.
- `runbooks/go-live-readiness.md` — the audit this batch acts on. **Correct its
  SOS row**: the row says "no SOS"; the truth is a button that falsely claimed
  success.
- This file.

---

## 5. Success condition — honest status

> *A driver can register, submit documents, register a vehicle, receive
> approval, accept a ride, complete it, and the rider can see the correct fare
> and access working safety controls.*

| Step | Status |
|---|---|
| Driver registers (OTP) | Was already working; live QA login re-confirmed |
| Submits documents | **Code complete, device-UNVERIFIED** — needs a camera |
| Registers a vehicle | **Code complete, device-UNVERIFIED** |
| Receives approval | Console path exists and is multi-country correct; end-to-end with a real upload **UNVERIFIED** |
| Accepts a ride | Pre-existing; **UNKNOWN on Android 14** until the foreground-service fix is seen on a device |
| Completes it | Pre-existing, backend-verified; device **UNVERIFIED** |
| Rider sees the correct fare | **Fixed and unit-tested**; verified against the live QA payload |
| Rider safety controls work | **Built and unit-tested**; the SOS reaching ops and the contact SMS are **UNVERIFIED** on a device |

**The success condition is not met, and cannot be met from this machine.** Every
remaining gap is a device or an owner action, not code. Per the instruction, no
claim of full success without device and QA evidence.

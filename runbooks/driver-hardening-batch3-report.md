# Canonical driver hardening + Android release preparation — batch 3

**Date:** 2026-10-16
**Scope:** canonical driver app hardening, email delivery, mobile auth
verification against server contracts, Android release readiness, Maps key
re-audit.
**Nothing was pushed, merged or deployed.** All work is on feature branches on
this machine. No Railway configuration was touched, no payments enabled, no
Android application published.

---

## 1. Branches and commits

| Repo | Branch | Base | Commits |
|---|---|---|---|
| `SaaradhiGo-driver` | `feat/driver-dev-hardening` | `origin/dev` @ `2252c2d` | `ea370d3`, `e66b3c0`, `c4abc78`, `b17d26b` |
| `SaaradhiGo-mobile` | `feat/android-pilot-batch1` | — | `7dccffd`, `463baa6` (this batch) |
| `SaaradhiGo-backend` | `feat/android-pilot-batch1` | — | `6eba956` (this batch) |
| `SaaradhiGo-docs` | `docs/india-launch-readiness` | — | this report + two runbook updates |

The driver branch was created from `origin/dev` — the canonical app — and no
history from `feat/driver-app-rebuild` was merged. The six agreed improvements
were ported as *source*, adapted to the canonical app's structure.

### What was ported from the rebuild lineage

| Item | Commit | Note on adaptation |
|---|---|---|
| CI workflow | `e66b3c0` | Firebase/push notes dropped (canonical app has no Firebase); a named Android-gate step added |
| Secure release signing + build guard | `e66b3c0` | Rewritten onto the canonical Gradle: Java 11, `kotlin-android`, canonical namespace, no Maps placeholder |
| Signing configuration test | `e66b3c0` | 7 tests; structural assertions read the file with comments stripped |
| Manifest foreground-location test | `e66b3c0` | 8 tests; retargeted from `flutter_background_service` to geolocator, plus an assertion that no Maps key is needed |
| Email/password login UI | `b17d26b` | Rebuilt as a second tab on the existing screen; the mobile tab is byte-identical in behaviour |
| Server-generated auth contract tests | `b17d26b` | 18 tests driving the canonical `AuthRepository` and the real `AuthInterceptor` |

The manifest **configuration** needed no change: `origin/dev` already declared
`foregroundServiceType="location"` and all three location grants. Nothing held
it that way, which is what the test now does.

### Three pre-existing gaps closed (`ea370d3`)

- **Bundled `.env` asset.** `.env` was a declared Flutter asset, so it shipped
  inside the APK and was extractable with `unzip`; worse, a missing `.env` in a
  release build silently fell back to the dev API host. Replaced with
  `--dart-define` resolution, a dev-only fallback, and a release-build
  misconfiguration check.
- **Duplicate incentives endpoint.** Two constants and a 404-retry meant every
  incentives read cost a guaranteed 404 first. One constant, one request.
- **`lib_backup/`.** 53 files deleted after verifying no references from
  `lib/`, `test/` or `integration_test/` and that live equivalents exist.

---

## 2. Screens and functionality preserved

| Check | Result |
|---|---|
| Files deleted vs `origin/dev` | **only** the 53 under `lib_backup/` |
| Screens under `lib/` | 35 before, 35 after |
| Dart files under `lib/` | 257 before, 257 after |
| Feature directories | all 13 intact: auth, dashboard, documents, earnings, incentives, notifications, onboarding, profile, splash, support, trip, trip_history, vehicle |
| Files changed under `lib/` | 12, all accounted for below |

The 25 `_screen.dart` files that disappear from a whole-repo count were all
inside `lib_backup/`.

Changed: `main.dart`, `core/config/feature_flags.dart`,
`core/network/{api_endpoints,auth_interceptor,dio_client,environment_config}.dart`,
`core/storage/secure_storage_service.dart`,
`features/auth/domain/auth_repository.dart`,
`features/auth/presentation/login_mobile_screen.dart`,
`features/auth/presentation/providers/{auth_notifier,auth_state}.dart`,
`features/incentives/data/data_sources/incentives_remote_data_source.dart`.

Onboarding, KYC, dispatch, navigation, trip history, incentives and support
code is untouched apart from the single incentives data-source fix.

---

## 3. Test results

| Suite | Result |
|---|---|
| Canonical driver — `flutter analyze` | clean |
| Canonical driver — `flutter test` | **343 passed** (290 at branch point, 53 new) |
| Driver Android gates (`release_signing` + `android_manifest`) | 15 passed |
| Driver auth (contract + session revocation + login tabs + notifier) | 40 passed |
| Rider — `flutter analyze` | no errors (39 pre-existing infos) |
| Rider — `flutter test` | **144 passed**, 7 skipped |
| Rider Android gates | 13 passed |
| Rider auth (contract + session revocation + login tabs + provider) | 37 passed |
| Backend — full `pytest` | **1295 passed**, 45 skipped, **9 failed** |
| Backend auth contract capture + email safety + receipt delivery | 34 passed |

**The 9 backend failures are pre-existing and Redis-dependent** (7 durable
dispatch, 2 executive revenue). Confirmed by stashing this batch's changes and
re-running them on `HEAD`: the same 9 fail identically. No Redis is available
on this machine; CI has one.

**The 7 rider skips are pre-existing** `skip: true` markers in
`personal_info_screen_test.dart` (4) and `profile_tab_test.dart` (3) — rider
profile UI paths that no test covers. They belong on the device checklist.

### Defect-catching verified, not assumed

Each new guard was run against the code it is meant to catch:

| Guard | Evidence |
|---|---|
| Driver signing test | 3 of 7 fail against the previous `build.gradle.kts` |
| Driver manifest test | the location test fails when `foregroundServiceType` is removed |
| Driver session revocation (10 tests) | 7 of 10 fail against the previous `SecureStorageService` |
| Rider session revocation (9 tests) | 4 of 9 fail with the read gate removed |
| Receipt email delivery (12 tests) | 5 of 12 fail against the previous `_send_receipt_email`, and the structural test names the offending line |
| Signing guard, against real Gradle | `flutter build apk --release` refused in both apps with the guard's own message |

---

## 4. Email delivery

Production-compatible and routed through one gate. `base/settings.py` already
read `EMAIL_BACKEND`/`EMAIL_HOST`/`EMAIL_PORT`/`EMAIL_HOST_USER`/
`EMAIL_HOST_PASSWORD`/`EMAIL_USE_TLS`/`EMAIL_TIMEOUT`/`DEFAULT_FROM_EMAIL` from
the environment, which is what AWS SES SMTP needs.

| Message | Path | Allowlist applies |
|---|---|---|
| Password reset | `auth_email.send_password_reset` | yes |
| Email verification | `auth_email.send_email_verification` | yes |
| Account invitation | `auth_email.send_account_invite` | yes |
| **Trip receipt** | `receipts._send_receipt_email` → `auth_email.send_message` | **yes — new in this batch** |

`servers/ride/receipts.py` previously built its own `EmailMultiAlternatives`
and sent it, so the one message type that reaches *customers* answered to
neither `delivery_is_configured()` nor `AUTH_EMAIL_ALLOWLIST`. "An empty QA
allowlist blocks all outbound email" was therefore false for receipts: QA
pointed at real SES, with a real address in a restored dump, would have emailed
that person a GST receipt for a test trip. `base/auth_email.py` is now the only
module in the codebase that calls `message.send()`, and a test enforces that by
scanning the repo for any other sender.

Safeguards preserved and proven: production is unrestricted; non-production
with a delivering backend is allowlist-only; an **empty** non-production
allowlist blocks everything, receipts included; console and locmem stay exempt
so local work and the suite function. A deployment with no `DEFAULT_FROM_EMAIL`
now reports `undeliverable` at ERROR level in the log stream instead of writing
a reason into `Receipt.send_failure_reason` where nobody looks.

**No production credentials were configured and no real customer email was
sent.** `base/settings_ci.py` gained a test-only `DEFAULT_FROM_EMAIL`
(`no-reply@ci.saaradhigo.test`); Django swaps in locmem under test, so nothing
leaves the machine.

### Owner actions still required

1. Verify an SES sender domain and set `DEFAULT_FROM_EMAIL`, `EMAIL_HOST`,
   `EMAIL_HOST_USER`, `EMAIL_HOST_PASSWORD` per environment.
2. Set `AUTH_EMAIL_ALLOWLIST` on QA **before** pointing it at SES. Leaving it
   empty is safe — it blocks everything.
3. SES starts in sandbox mode; request production access when real sending is
   wanted. Details in `runbooks/ses-email-setup.md`.

---

## 5. Mobile authentication, verified against the server's own contracts

`contracts/auth_contracts.json` is generated by
`tests/test_auth_contract_capture.py` from real responses produced by the real
views, and copied into both apps. Neither app's tests can be "fixed" by editing
a fixture to match the app — which is precisely how the rebuilt driver app
shipped reading `data['access']` while the server sends `token`.

Covered in both apps: OTP login, email + password login, token persistence,
refresh (driven through the real interceptor, not a mock), logout, password
reset, and the RBAC refusals. The driver suite also asserts the endpoint
constants against the paths the server recorded, and that the OTP and email
envelopes remain identical.

### Secure-storage failures can no longer restore a logged-out session

Both apps had a hole, in mirror-image forms.

**Driver.** `clearTokens()` was three bare awaits. A throw from the first
delete left the refresh token and profile on disk, propagated out of
`AuthRepository.logout()` into `AuthNotifier.logout()` where the state change
sat on the line *after* the await and never ran — the driver pressed Log out,
saw an error, stayed signed in — and the next cold start restored the session.
The same three lines run in `AuthInterceptor._refreshTokens()` when the server
*rejects* a refresh token, so a throw there kept alive a session the server had
already repudiated.

**Rider.** `TokenStore.clear()` already caught the throw so the prefs copies
were removed. The secure copy survived, and `_read` consults secure storage
first — so the next launch read a token the rider had logged out of.

Both now set a revocation tombstone *before* attempting any delete, attempt
each key independently, fall back to `deleteAll()`, and **return null from
every read while the tombstone is set**. The gate on the read is what closes
it, which is why it covers the interceptor without the interceptor knowing. The
tombstone lives in memory and in SharedPreferences — a different store from the
one failing — holds no credential, and is lifted by a successful login so a
later sign-in works normally. An unreadable tombstone counts as revoked: a
store the app cannot consult is not consent to use a credential.

Residual risk, stated in both source files: if the secure store *and*
SharedPreferences are both unwritable, the tombstone cannot outlive the
process. There is then nowhere left to record the fact; within a running
process the guarantee is unconditional.

### Three further defects found while wiring email login

- `DioClient` read `data['message']`, which this API never sends — every error
  envelope is `{"status": "error", "error": {"message": ...}}`. So every
  refusal reached the driver as a hardcoded "Authentication failed" instead of
  "Invalid email or password." or the MFA prompt.
- `AuthInterceptor` excluded only `/auth/login/` from refresh-on-401, so a
  rejected **email** login ran the refresh path, cleared the session and
  emitted `sessionExpired` — telling a driver who mistyped their password "Your
  session has expired". All four unauthenticated auth paths are now excluded.
- `AuthRepository.logout()` only cleared local tokens, leaving the refresh
  token valid for its full lifetime; a copy taken off a lost phone kept minting
  access tokens. It now calls `POST /auth/logout/` first, best-effort with a
  5-second timeout so a driver with no signal can still log out.

Also fixed: all 22 state writes in `AuthNotifier` happen after an await or a
3-second timer and wrote to `state` unguarded. Riverpod 3 throws `Cannot use
the Ref ... after it has been disposed`, so leaving the login screen while an
error message was held crashed the app. They now go through a `_setState` that
checks `ref.mounted`.

---

## 6. Android signed-build readiness

**The local release-build blocker is gone.** The earlier report that this
machine could not produce a release build was true at the time and is no longer:
the missing pieces were SDK components, which the first attempts installed
(NDK r27, Platform 35, CMake 3.22.1). Full evidence in
`runbooks/android-signing-setup.md` §3. In summary: both apps refuse an
unsigned release build with the guard's own message, both build successfully
with `-PallowDebugSigning=true`, `flutter build appbundle` is refused too, and
`apksigner` confirms the opt-out artifact is debug-signed. Both artifacts were
deleted.

Both keystores were verified readable with their stored passwords and expected
aliases, valid to 2054 (SHA-1s in the signing runbook).

### The one thing blocking signed release artifacts

> **Neither keystore is backed up.** `C:\Users\srini\SaaradhiGo-signing\`
> contains both `.jks` files and `credentials.txt`, and a search of this
> machine finds **no second copy anywhere**. The passwords sit in the same
> directory as the keys, so one compromise yields both.

The instruction for this batch was to verify the backups *before* producing
release artifacts. They do not exist, so **no signed release artifact was
produced**. This is an owner action and it is the gate: a password manager or
encrypted vault entry holding both `.jks` files and `credentials.txt`, reachable
by at least two people.

### CI validation added

Both repos' `flutter` workflow now runs, as a separately named step, the
signing-configuration and Android-14 location tests — so a regression reads as
"the Android release configuration regressed" rather than one line among
hundreds. The `release-config` job proves the guard works **with no secret at
all**: an unsigned release build must fail with the guard's message, and the
documented opt-out must still succeed.

### Before the first Play upload

- Play needs an **`.aab`**, not an `.apk` (App Bundle only for new apps since
  August 2021).
- Both APKs are ~60 MB universal builds. `--split-per-abi`, or the AAB, cuts
  that to roughly a third — worth doing before asking drivers on metered data
  to download it.
- The driver `applicationId` is still the generated
  `com.saaradhigo.saaradhigo_driver` and **can never change after the first
  upload**. Decide now whether the pilot ships as `com.saaradhigo.driver`.
- The rider release build must pass `-Pmaps.apiKey=…` or its manifest
  placeholder resolves empty. Nothing currently reads that key (see §7), so
  this is a latent trap rather than a live one.

---

## 7. Maps key — re-audited

No mobile app depends on a Google Maps key: neither `pubspec.yaml` nor either
lockfile contains `google_maps_flutter`, and both apps render with
`flutter_map` over OSM tiles. The exposed key is in **no file at HEAD of any
repo**; it survives in two rider commits, which is enough to require rotation
but means no build reads it. The canonical driver app is now *tested* to
contain neither `AIza` nor `com.google.android.geo.API_KEY`, so it cannot
quietly acquire the dependency.

**Live usage and current restrictions could not be checked from here** — that
needs Google Cloud Console access. The key was also deliberately not probed
with a test request: that would send a production credential to a third party
and put a line in the owner's billing to learn what one console page states.

**The one thing that decides whether revocation is safe** is whether the
backend's deployed `GOOGLE_MAPS_API_KEY` is the same key. It is read from the
environment with an empty default and appears in no source file, so it cannot
be determined from the repos. `runbooks/maps-key-rotation.md` §2a gives a
fingerprint comparison that answers it without anyone pasting a key, and lists
the four backend routes (`/ride/maps/geocode`, `place-details`,
`reverse-geocode`, `directions`) that fail if the key is revoked while still in
use — which would leave a rider unable to enter a destination.

Not done, per instruction: nothing was revoked or rotated.

---

## 8. Remaining device-testing blockers

Everything below is **UNKNOWN**, not PASS. No Android device or emulator is
available in this environment, so nothing here has run on a phone.

| # | Blocker | Why it needs a device |
|---|---|---|
| 1 | **Keystore backup** | Gates producing any signed artifact at all |
| 2 | Signed APK/AAB install and upgrade-over-install | Only a device proves the signature chain works |
| 3 | Android 14+ foreground location while on duty | `startForeground()` for a location service is enforced at runtime; the manifest test proves the declaration, not the behaviour |
| 4 | Sustained GPS tracking through screen-off and app-background | The original pilot blocker; needs hours on a real phone |
| 5 | Email + password login against QA | Needs `DEFAULT_FROM_EMAIL` and an allowlist entry on QA first |
| 6 | Receipt email arriving, with its PDF | Needs SES configured and an allowlisted QA recipient |
| 7 | Logout, including with secure storage degraded | The fault is simulated in tests; a real Keystore failure cannot be |
| 8 | Rider profile and personal-information screens | 7 skipped tests cover none of it |
| 9 | Driver onboarding and KYC document upload end to end | Presigned upload against QA storage |
| 10 | Dispatch, accept, navigate, complete, receipt — one full trip | The only test that matters |

---

## 9. Exact requirements for the next controlled QA release

In order. Items 1–4 are owner actions and nothing else can start without them.

1. **Back up both keystores and `credentials.txt`** to a password manager or
   encrypted vault, reachable by two people. *Gates everything below.*
2. **Decide the driver `applicationId`** — keep `com.saaradhigo.saaradhigo_driver`
   or change to `com.saaradhigo.driver`. Irreversible after the first upload.
3. **Configure SES for QA:** verified sender domain, `DEFAULT_FROM_EMAIL`,
   `EMAIL_HOST`, `EMAIL_HOST_USER`, `EMAIL_HOST_PASSWORD`, and
   `AUTH_EMAIL_ALLOWLIST` set to the QA addresses **before** the first send.
4. **Fingerprint the deployed `GOOGLE_MAPS_API_KEY`** and decide whether the
   exposed key is rotated or simply revoked.
5. **Approve pushing the four feature branches.** CI has never run on any of
   this; the three jobs per app are new. Nothing is merged to `dev` or `main`.
6. **Create `android/key.properties` in each app** from `credentials.txt` (both
   are gitignored), or set the four `ANDROID_*` CI secrets per repository.
7. **Build the QA artifacts** with the dart-defines pointing at QA:
   `flutter build apk --release --split-per-abi
   --dart-define=API_BASE_URL=… --dart-define=WS_BASE_URL=…` for the driver,
   plus `-Pmaps.apiKey=…` for the rider if a key is in play.
8. **Apply migrations to QA** — 7 are already applied; confirm no new ones.
9. **Run the device checklist** in `runbooks/android-device-test-checklist.md`
   on the QA phones, covering blockers 2–10 above.
10. **Do not enable live payments.** Unchanged from every previous batch.

---

## 10. What was deliberately not done

- Nothing pushed, merged, or deployed; no Railway configuration read for
  secrets or modified.
- No signed release artifact produced — gated on the keystore backup.
- No production email credentials configured; no real customer email sent.
- No Maps key revoked, rotated, or probed.
- No Android application published.
- The rider app's vestigial `com.google.android.geo.API_KEY` manifest entry was
  left in place: removing it is correct but is rider cleanup, not driver
  hardening, and it is inert.

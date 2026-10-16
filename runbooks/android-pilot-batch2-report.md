# Android pilot — batch 2 report

Written for: the owner, reviewing before anything is merged or deployed.

**Nothing merged to `dev` or `main`. No Railway build or deployment. No
service deleted. No production change. Live payments still off.**

| Repo | Branch | Commit | Pushed |
|---|---|---|---|
| SaaradhiGo-backend | `feat/android-pilot-batch1` | `bba8f09` | yes |
| SaaradhiGo-mobile (rider) | `feat/android-pilot-batch1` | `0bbb765` | yes |
| SaaradhiGo-driver | `feat/android-pilot-batch1` | `d222301` | yes |

Verified safe to push first: the five repo-connected Railway services watch
`SaaradhiGo-backend@dev` (four of them) and `SaaradhiGo-web@develop` (one).
**No service watches a feature branch, and the two mobile repos have no
Railway service at all.** Only two environments exist (QA, production), so
there are no PR-triggered environments either.

---

## 1. Railway auto-deploy — every service

All five repo-connected services now have
`watchPatterns: [".railway-autodeploy-disabled/**"]`, a path that never
exists, so a push matches nothing and no build is queued.

| Service | Env | Source | Auto-deploy | Verified |
|---|---|---|---|---|
| `backend` | QA | `SaaradhiGo-backend@dev` | **suppressed** | `get-service-config` |
| `celery` | QA | `SaaradhiGo-backend@dev` | **suppressed** | `get-service-config` |
| `alluring-happiness` | QA | `SaaradhiGo-backend@dev` | **suppressed** | `get-service-config` |
| `SaaradhiGo-backend` | production | `SaaradhiGo-backend@dev` | **suppressed** | `get-service-config` |
| `web` | QA | `SaaradhiGo-web@develop` | **suppressed** | `get-service-config` |
| `Postgres` | QA | image `postgres-ssl:18` | n/a — no GitHub trigger | — |
| `Redis` | QA | image `redis:8.2` | n/a — no GitHub trigger | — |

**No redeploy or restart was triggered.** `describe-environment` after the
change shows every service's latest deployment unchanged (08:55 UTC for the
three backend services, September for `web`, `Postgres`, `Redis`).

### Be precise about what this is

A watch pattern is the **Railway-supported deploy trigger setting**, and it is
the only way to stop auto-deploys through the API without a build. It is not a
disconnect:

- The repo link, branch and build config are all untouched, so nothing has to
  be reconnected later.
- A **manual** redeploy still works — which is what you want for the deploy
  gate.
- A future engineer could clear the pattern in the dashboard without realising
  what it was for. The pattern is named to make that obvious.

The alternatives both need your approval because they build or delete:
`connect-service-source` with `commitSha` pins the service and **builds
immediately**; applying the staged deletions removes two services.

### The two staged deletions are untouched

`alluring-happiness` (QA) and `SaaradhiGo-backend` (production) each still
carry a **staged, uncommitted deletion** from 2026-09-23. I did not commit
them — deletion is excluded, and `accept-deploy` on QA would apply a patch
flagged `destructive: true`. Both are zero-variable shells that cannot boot;
`alluring-happiness`'s last deployment is `FAILED`, from one of my own earlier
pushes to `dev`.

---

## 2. Driver branch reconciliation — STOPPED, needs your decision

**This is not a 19-ahead/20-behind divergence. The two branches have no common
ancestor.**

```
git merge-base HEAD origin/dev   ->  exit 1 (no common ancestor)

feat/driver-app-rebuild   root df72622 "scaffold SaaradhiGo Drive app shell"   86 .dart files
origin/dev                root 2f332fd "First"                               363 .dart files
shared filenames: 4  (main.dart, api_endpoints.dart, location_service.dart,
                      auth_repository.dart)
```

Two different driver applications live in one repository. A merge would need
`--allow-unrelated-histories` and would produce a repo containing both apps,
with guaranteed conflicts in `main.dart` and `api_endpoints.dart` and no
working result. **A rebase would be worse.** So I did neither.

### What each side actually has

| | `feat/driver-app-rebuild` (+ batches 1–2) | `origin/dev` |
|---|---|---|
| Architecture | feature/domain/data, Riverpod `Notifier` | provider-based, freezed models |
| Tests | **142**, analyze clean | **none** |
| CI | analyze + test + debug APK + release-config | **no workflows at all** |
| KYC upload | built in batch 1 | **already present** — `core/uploads/presigned_upload_service.dart`, document vault, onboarding screens, verification status |
| Android 14 foreground service | fixed in batch 1 | **already correct** |
| Release signing | fixed in batch 1 | **debug key** — same defect |
| Committed API key | none | none |
| Login | **was broken**, fixed in batch 2 (see §3) | unverified |

### What this means, plainly

- My audit's headline P0 — "a driver cannot be onboarded, no UI anywhere" —
  was **true of the branch I audited and not true of `origin/dev`**. I
  reported a branch-specific finding as a product fact. The readiness report
  is corrected.
- Batch 1's KYC upload work **duplicates capability that already exists** on
  `origin/dev`, in a different architecture. If `dev`'s app ships, most of it
  is wasted (the signing fix and the CI still apply).

### The decision is yours, and it is not a merge

1. **Ship the rebuild** (`feat/driver-app-rebuild` + batches): tested, CI'd,
   now actually able to log in — but 86 files against `dev`'s 363, so feature
   parity needs an audit before you can trust it.
2. **Ship `dev`'s app**: already has KYC and more screens — but zero tests,
   zero CI, debug signing, and its login path is unverified against the real
   API.
3. **Pick one as canonical and archive the other**, the way
   `operations-console-decision.md` settled the two consoles.

I recommend (3) with a short parity audit before choosing, and I have not
touched either branch's history.

---

## 3. A P0 found while adding email login: the driver app could never log in

`auth_repository_impl.verifyOtp` read `data['access']` and `data['refresh']`.
The backend sends `token` and `refresh_token`:

```
servers/auth_user/views.py  login    -> {'token', 'refresh_token', 'user'}
servers/auth_user/views.py  refresh  -> {'token', 'refresh_token'}
```

So login always threw *"Login succeeded but no token was returned"* and token
refresh always failed silently — every session would have died at the
15-minute access-token lifetime even if login had worked.

**Nothing caught it.** `flutter analyze` cannot see an API contract, and the
existing tests built their own fixtures using the same wrong key names the
code used, so the tests and the code agreed with each other and both
disagreed with the server. That is the lesson worth keeping: a test written
from the implementation proves only that the implementation is
self-consistent.

Fixed, with the real envelope copied from the backend view into the test.

---

## 4. Email + password authentication

Audited first. Most of what this needed already existed and was reachable only
from the operator console: Django's PBKDF2 via `AbstractUser`, two
phone+password login paths, `login_guard` (8 per identity / 15 per IP / 15-min
lockout, failing open on a cache outage by documented choice), `ops_mfa`
(django-otp TOTP + static recovery), `admin_audit`, and per-phone OTP
throttles. **No hashing, token algorithm or cryptography is written in this
change.**

What was genuinely missing, and is now built:

| Gap | Resolution |
|---|---|
| `email` was a non-unique nullable `CharField` — decoration, not identity | `EmailField` + a DB unique constraint on `LOWER(email)` where present; NFKC-normalised and lower-cased before comparison |
| No password login for riders/drivers | `POST /auth/email/login/`, same token envelope as `/auth/login/` |
| No set/change/forgot/reset | Four endpoints on Django's `PasswordResetTokenGenerator` — single-use, 2-hour expiry (cut from Django's 3-day default) |
| No proof of email ownership | Verification endpoints with a **different `key_salt`**, so a verification token cannot be replayed as a reset token |
| No way to create an account without handing out a password | `POST /auth/admin/invite/` — nothing generated, returned, logged or stored |

### The security decisions worth your attention

- **One refusal for every credential failure**, including the `issue` field
  that reaches the client. An unknown address also pays the cost of a password
  hash, so it does not return measurably faster and become a timing oracle.
- **Operators must present TOTP** on email login too. A wrong code counts
  toward the lockout — six digits is brute-forceable in a way a password is
  not.
- **Email login never creates an account.** A phone number proves possession
  of that phone; an address proves nothing until verified.
- **Reset requires a verified address.** Otherwise anyone who could set an
  unverified address on an account they control could pivot into it.
- **Forgot-password always answers 200** with one message. For a ride-hailing
  platform, confirming that an address belongs to a driver is itself a
  disclosure.
- **Linking an email never moves authority.** `link_email` snapshots `role`,
  `is_staff`, `is_superuser` and `ops_role` and raises if any changed.
- **Only an `ops_role=admin` operator may create an operator**, and only with
  MFA already cleared on the session — a compromised support account cannot
  mint an admin. `is_superuser` is simply not read. An existing address or
  phone is **refused, never re-pointed**.
- **The migration refuses to run on duplicate addresses**, reporting account
  ids and never addresses, because choosing which account keeps an address is
  a support decision.

---

## 5. Email delivery is not configured — and the code says so

There was **no `EMAIL_*` setting anywhere**, so Django defaulted to SMTP on
`localhost:25`. Receipt email has been failing silently this whole time: the
Receipt row is written, `send_failure_reason` is set, and nobody watches that
column.

Settings now read `EMAIL_*` from the environment, and `base/auth_email.py`
**refuses to report success for a message that cannot have been sent** — it
logs `auth_email_undeliverable` at ERROR and returns False, and the invite
response tells the operator plainly. It never logs the address, because an
auth log listing addresses it failed to reach is an account list.

**Owner action required** on the `backend` service: `EMAIL_HOST`,
`EMAIL_PORT`, `EMAIL_HOST_USER`, `EMAIL_HOST_PASSWORD`, `EMAIL_USE_TLS`,
`DEFAULT_FROM_EMAIL`. SES is the natural choice (SMS already uses it) and
needs a **verified sender domain**. Applying Railway variables is a deploy, so
it waits for the gate. QA workarounds that need no email are in
`qa-test-accounts.md`.

---

## 6. Google Maps API key

Identified by fingerprint, never printed. Full procedure in
`maps-key-rotation.md`.

| Fingerprint | What | Action |
|---|---|---|
| `863b98dead568420` | Maps API key, committed since the initial commit in **two** places | **ROTATE — owner only** |
| `338d92da324c9d28` | Firebase Android key in `google-services.json` | Ships by design; verify restrictions, do not rotate |

Batch 1 removed it from `AndroidManifest.xml` and **missed a second copy** in
`ios/Runner/AppDelegate.swift`, which batch 2 removed. That file also
`import GoogleMaps` with no GoogleMaps pod, so it could never have compiled,
and the rider app uses `flutter_map` over OSM — the key was dead twice over.

Required restrictions are documented: Android apps (package + SHA-1) and
*Maps SDK for Android* only for app keys; API-restricted plus quota caps for
the server key. **Rotation is still outstanding — removing it from HEAD does
not rotate it.**

Also found: **QA has no `GOOGLE_MAPS_API_KEY` at all**, so every QA fare uses
the haversine fallback rather than Distance Matrix. Device testing on QA does
not exercise the real distance path.

---

## 7. Signing readiness

| | Rider | Driver |
|---|---|---|
| Reads `key.properties` | yes | yes |
| Reads CI env (`ANDROID_KEYSTORE_PATH`/`_KEY_ALIAS`/`_KEY_PASSWORD`/`_STORE_PASSWORD`) | yes | yes |
| Unsigned release build **refuses** | yes | yes |
| Logged opt-out for local release runs | `-PallowDebugSigning=true` | same |
| Debug builds unaffected | yes | yes |
| Keystore / password committed | none | none |
| Gitignored (`key.properties`, `*.jks`, `*.keystore`) | yes | yes |
| Test asserting no stray keystore in the tree | yes | yes |
| CI validates the guard without secrets | `release-config` job | `release-config` job |

**Blocked on you:** no keystore exists. I have not generated one — production
signing credentials are not mine to create. `origin/dev`'s driver app has the
same debug-signing defect and would need this fix ported if it is the app that
ships.

---

## 8. Test evidence

```
BACKEND  feat/android-pilot-batch1 @ bba8f09
  ruff check base/ servers/ tests/ qa/        All checks passed!
  makemigrations --check --dry-run            No changes detected
  pytest                                      1261 passed, 9 failed, 45 skipped
      the 9 are the known Redis-dependent locals (7 durable_dispatch,
      2 executive_revenue); identical at the branch point, green in CI
  tests/test_email_password_auth.py            62 passed   (new)
  tests/test_otp_cache_roundtrip.py            10 passed

RIDER    feat/android-pilot-batch1 @ 0bbb765
  flutter analyze --no-fatal-infos            exit 0
  flutter test                                123 passed, 7 skipped
      test/login_tabs_test.dart                 9 passed  (new)

DRIVER   feat/android-pilot-batch1 @ d222301
  flutter analyze --no-fatal-infos            "No issues found!"  exit 0
  flutter test                                142 passed
      test/auth_token_contract_test.dart        9 passed  (new)
      test/login_tabs_test.dart                 9 passed  (new)

LIVE QA (read-only, no trip created, no SMS, no charge)
  POST /auth/otp/   rider +919999000001        200, bypass fired, no SMS
  POST /auth/login/ rider                      200, 256-char token
  POST /auth/otp/   driver +919999000002       200, bypass fired, no SMS
  POST /auth/login/ driver                      200, role=driver, 256-char token
```

QA OTP authentication is **healthy** — both test accounts round-tripped, and
the transient failure seen in batch 1 did not recur.

---

## 9. Device testing blockers

Unchanged and unchangeable from here: `flutter build apk` fails on this
machine (symlink/Developer Mode), no emulator, no device, no macOS. CI builds
debug only, so **no signed release APK exists anywhere yet.**

Everything device-dependent is in `android-device-test-checklist.md`, now
covering all eight requested areas plus authentication. The items that are
**first-time verifications rather than regression checks**, because the code
path has never worked:

- driver OTP login and token refresh (§3 above);
- the rider SOS reaching ops and the emergency-contact SMS (the rider app had
  no SOS call before batch 1);
- driver document upload end to end on the rebuild branch.

---

## 10. Owner actions required

1. **Decide the driver app question** (§2). Everything else about the driver
   app is downstream of it.
2. **Android keystore** for both apps, as CI secrets.
3. **Rotate the Maps key** `863b98dead568420` and apply the documented
   restrictions.
4. **Email delivery**: SES SMTP credentials and a verified sender domain.
5. **A QA `GOOGLE_MAPS_API_KEY`**, with quota caps, if you want device testing
   to exercise real road distances.
6. **MFA policy:** `OPS_MFA_ENFORCED` is off outside enforced environments, so
   an operator without a TOTP device can currently sign in on QA. Confirm
   whether QA should enforce it before the pilot.
7. The two staged service deletions, when you want them applied.
8. `EC2_HOST_KEY`, or a decision to drop the EC2 deploy job — it fails on
   every push and will block QA deploys if `checkSuites` is ever enabled.

---

## 11. Exact next release gate

**Gate: a signed release APK of both apps, installed on an Android 14+ device,
passing §§1–9 of the device checklist against QA.**

In order:

1. You decide §2 (driver app) and supply the keystore (§10.2).
2. Open PRs for the three branches — **do not merge**. Rider and driver CI run
   on every branch, so analyze, tests, the debug APK and `release-config` are
   already running. Backend CI only triggers on `dev`, so its branch needs a
   PR to run at all.
3. Add the signing secrets; CI produces a signed release APK.
4. Configure email (§10.4) so QA email accounts and receipts work.
5. Merge the three branches to `dev` **together**, in one batch. With
   auto-deploy suppressed, this builds nothing.
6. **Manually** redeploy `backend` and `celery` on QA — one deploy, deliberate
   and attributable.
7. Run the device checklist. Record PASS/FAIL per line with device and build.
8. Only then is a go/no-go on the controlled pilot a real question.

**No go-live readiness is claimed. The Android device tests have not run.**

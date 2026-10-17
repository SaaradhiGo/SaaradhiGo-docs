# CI and Android release preparation — batch 4

**Date:** 2026-10-17
**Scope:** push the four feature branches, get CI green on all of them,
settle the driver application ID, prepare the AAB and QA APK build
configuration, prepare SES QA configuration, verify the Maps key
dependencies, and update the device checklist.

**Not done, per the restrictions:** nothing merged to `dev` or `main`, no
Railway deployment, no production payment enablement, no Play upload, no real
customer SMS or email, no secret printed or committed, no signed artifact
produced.

---

## 1. Pushed branches and commits

Pushed after confirming no branch can trigger a Railway deployment (§2).

| Repo | Branch | Head | New this batch |
|---|---|---|---|
| `SaaradhiGo-driver` | `feat/driver-dev-hardening` | `df54387` | `c60796d`, `0a493c1`, `df54387` |
| `SaaradhiGo-mobile` | `feat/android-pilot-batch1` | `1e5f712` | `1e5f712` |
| `SaaradhiGo-backend` | `feat/android-pilot-batch1` | `e5b24dd` | `9b576e2`, `e5b24dd` |
| `SaaradhiGo-docs` | `docs/india-launch-readiness` | — | this report + 4 runbook updates |

`feat/driver-dev-hardening` was a new remote branch; the other three were
fast-forwards. No force push, no rebase, no merge.

### Why no push could deploy — three independent reasons

Checked read-only against the Railway API before pushing; nothing was
modified.

| Service | Env | Repo watched | Branch | Watch patterns |
|---|---|---|---|---|
| `backend` | QA | `SaaradhiGo-backend` | `dev` | `.railway-autodeploy-disabled/**` |
| `celery` | QA | `SaaradhiGo-backend` | `dev` | same |
| `alluring-happiness` | QA | `SaaradhiGo-backend` | `dev` | same (staged for deletion) |
| `web` | QA | `SaaradhiGo-web` | `develop` | same |
| `SaaradhiGo-backend` | production | `SaaradhiGo-backend` | `dev` | same (staged for deletion) |

1. No service watches `SaaradhiGo-driver`, `SaaradhiGo-mobile` or
   `SaaradhiGo-docs` at all.
2. The services that watch `SaaradhiGo-backend` watch the `dev` **branch**;
   none of these pushes touched it.
3. Every service's watch pattern is `.railway-autodeploy-disabled/**`, a path
   that does not exist, so even a `dev` push would match nothing.

`checkSuites: false` on every service, so CI results do not gate or trigger
anything either. The two staged service deletions remain staged and were not
applied.

**The backend repo also has an EC2 deploy job** (`Test and deploy` →
`deploy`), which is gated on
`github.event_name == 'push' && github.ref == 'refs/heads/dev'`. A feature
branch push and a pull request both run the test jobs and skip it. Confirmed
skipped on the run below.

---

## 2. CI results

All four branches, all jobs, green.

| Repo | Workflow | Jobs | Result |
|---|---|---|---|
| `SaaradhiGo-driver` | `flutter` | analyze-and-test, build-android, release-config | ✅ all green (first run this branch ever had) |
| `SaaradhiGo-mobile` | `flutter` | analyze-and-test, build-android, release-config | ✅ all green |
| `SaaradhiGo-backend` | `Test and deploy` | lint, test, postgres-concurrency | ✅ green; `deploy` **skipped** |
| `SaaradhiGo-docs` | — | — | no workflows in this repo |

**No CI failure was caused by these branches.** Two things were fixed to get
there, and one pre-existing failure was diagnosed and left alone.

### The backend branch had no CI at all

`Test and deploy` triggered only on `dev` — which is also the branch that
deploys. So the only way to discover that a change broke ruff, the suite or
the Postgres concurrency check was to push it to the deploying branch.
`feat/**` and `fix/**` now run the three gates, plus `workflow_dispatch` for a
manual re-run. The `deploy` job is untouched and still cannot fire from a
feature branch.

Result on `feat/android-pilot-batch1`: lint 22s, test 2m18s,
postgres-concurrency 3m54s, deploy skipped. The CI suite is greener than local
because CI has Redis; see §7.

### `dev` has been red for 20 hours, and not because of code

`Test and deploy` on `dev` has failed its last four runs. In every one,
**lint, test and postgres-concurrency passed** and only `deploy` failed, at
"Setup SSH" — the `EC2_SSH_KEY` / `EC2_HOST` secrets are missing or invalid.
So the backend has not actually been deploying to EC2, and the red X is
infrastructure, not code. Out of scope for this batch; worth a decision,
since a permanently red default branch trains people to ignore CI.

### The driver's formatting step

CI's formatting check is informational (`continue-on-error: true`) and was
reporting exit 1. The 9 files this branch touched are now `dart format`
output; the other 7 unformatted files under `lib/` and `test/` pre-date the
branch on `origin/dev` and were left alone — reformatting the tree is a
separate reviewable change, not something to fold into a hardening batch.

### The signing guard was proven in CI, not just asserted

From the driver's `release-config` log:

```
Release build refused: no Android signing material.
An unsigned release build was refused, with the expected message.
...
✓ Built build/app/outputs/flutter-apk/app-release.apk (62.9MB)
```

An unsigned release build failed with the guard's own words, and the
documented opt-out then succeeded — on a runner holding no secrets.

---

## 3. Android application ID recommendation

### Verified current state

| App | Android `applicationId` | iOS bundle id | Firebase Android client | FCM | Deep links | Play listing |
|---|---|---|---|---|---|---|
| Rider | `com.deeptrics.saaradhigo` | `com.example.vahango` | yes — project `saaradhigo-4f8ad` | real token registered | none | no evidence |
| Driver | `com.saaradhigo.saaradhigo_driver` | `com.saaradhigo.saaradhigoDriver` | **none** | **sends the literal `'dummy_device_token'`** | none (MAIN/LAUNCHER only) | no evidence |

- The rider's `google-services.json` registers exactly one Android client,
  `com.deeptrics.saaradhigo`, in Firebase project `saaradhigo-4f8ad`
  (project number 433870012396). Its `oauth_client` array is empty — no Google
  Sign-In client is tied to the package.
- The driver app has **no** `google-services.json`, no Firebase dependency in
  `pubspec.yaml`, and no deep-link intent filter. Nothing outside the repo is
  bound to its ID.
- Neither repo contains fastlane or Play metadata, and no backend code
  references either package name. No prior release registration was found.

### Recommendation: change the driver to `com.saaradhigo.driver`, now

**The rider's `com.deeptrics.saaradhigo` should not change.** It has a
Firebase Android client registered against it, and renaming means
re-registering that client and reissuing `google-services.json` for no
product benefit.

**The driver's should change, and this is the last moment it is free.** The
cost is zero today — no Firebase client, no FCM token mapping, no OAuth
client, no deep link, no Play listing — and after the first upload it is
permanent.

Why `com.saaradhigo.driver`:

1. **The current ID is an artifact, not a decision.** Flutter generated
   `com.saaradhigo.saaradhigo_driver` from the project directory name. It
   doubles the brand and carries an underscore, which is legal but
   unconventional and appears in the public Play URL.
2. **It matches the domain the platform actually runs on** —
   `api.saaradhigo.in`, `PLATFORM_CONTACT_DOMAIN = saaradhigo.in` — rather
   than the agency name in the rider's older ID.
3. **The runbooks already assume it.** `maps-key-rotation.md` §3 specifies the
   Android key restriction as package `com.saaradhigo.driver`. Keeping the
   generated ID means editing that and anything else written against it.
4. **It leaves `com.saaradhigo.rider` available**, so if the rider is ever
   re-registered the pair is symmetrical.

The alternative, `com.deeptrics.saaradhigo.driver`, is defensible — it matches
the rider's existing vendor prefix — but it entrenches an ID that was itself
not chosen deliberately, and it is longer in every place a user sees it.

**Scope note:** this was not applied. The brief for the previous batch was to
keep the canonical driver app based on `origin/dev` and to *recommend* the ID.
It is a one-line change to `android/app/build.gradle.kts` plus the iOS bundle
id for symmetry, and should be made in its own commit once approved.

### FCM: a real gap, surfaced by this check

The backend sends push via `firebase_admin`
(`servers/auth_user/services.py`), the rider registers a real FCM token, and
the **driver app sends the literal string `'dummy_device_token'`** at login.
So driver push does not work at all. Dispatch reaches a driver over the
WebSocket, which covers a foregrounded app; a ride offer to a backgrounded or
killed driver app cannot arrive. That is a pilot-scope decision, not a bug
introduced here, and it is a second reason to settle the ID now: adding FCM
later means registering a Firebase Android client against whichever ID is
final.

---

## 4. Keystore backup status

**Still not backed up. This is unchanged from batch 3 and it remains the
gate.**

| Check | Result |
|---|---|
| Both keystores present | ✅ `C:\Users\srini\SaaradhiGo-signing\` |
| Readable with stored passwords, expected aliases | ✅ verified, valid to 2054 |
| Second copy anywhere on this machine | ❌ none found |
| Off-machine backup | ❌ not verified — owner action |
| Passwords stored apart from the keys | ❌ `credentials.txt` sits beside both `.jks` files |

The build scripts now **refuse to produce a signed artifact** until
`SAARADHIGO_KEYSTORE_BACKUP_VERIFIED=yes` is set in the environment. That
turns the batch-3 runbook sentence into a gate that has to be cleared
deliberately, by someone who has done the backup.

---

## 5. Signed-build blockers

Only one, and it is the backup above. Everything else is ready and verified:

| Item | State |
|---|---|
| Local toolchain can build a release | ✅ both apps (NDK r27, Platform 35, CMake 3.22.1 now installed) |
| Signing guard refuses an unsigned release | ✅ locally and in CI, APK and AAB paths |
| Keystores readable | ✅ |
| Build configuration | ✅ `tool/build_release.sh` in both apps |
| QA release build can actually run | ✅ fixed this batch — see below |
| Artifact size | ✅ `--split-per-abi`: driver 22 MB, rider 24 MB (was ~60 MB universal) |
| `.aab` for Play | ✅ `play-aab` target; the guard covers `bundle` tasks |

### The QA acceptance build was impossible until this batch

Both apps refuse a **release** build pointed at a non-production host — the
guard that stops a store build talking to QA. A QA acceptance build is
*exactly* a release build pointed at QA, so the guard refused the one artifact
QA has to certify, and the alternative (certifying a debug binary) tests
different code: no minification, no shrinking, no release signing.

`ALLOW_NON_PRODUCTION_HOST` is a compile-time opt-in for that one case. It
does not weaken what it opens: a release build with **no** server configured
is still refused and is checked first, the flag cannot be acquired after the
build, and the environment label still reports non-production so the QA banner
and badge stay on every screen.

The driver app also had a related defect: `EnvironmentConfig.misconfiguration`
was **dead code** — its own comment claimed `main()` rendered it, and `main()`
never read it. So a release build with no `--dart-define` silently fell back
to the dev API. It now renders a "This build cannot be used" screen, and shows
a red `QA` corner banner on any non-production build.

---

## 6. SES and Maps configuration blockers

### SES

QA has **no email variables at all** — verified from the Railway service
variable names (no values requested): no `EMAIL_BACKEND`, `EMAIL_HOST`,
`EMAIL_HOST_USER`, `EMAIL_HOST_PASSWORD`, `DEFAULT_FROM_EMAIL` or
`AUTH_EMAIL_ALLOWLIST`. So QA cannot send anything today, and every receipt
and password-reset has been failing into `Receipt.send_failure_reason` and the
log.

The code side is ready: the allowlist is enforced for every outbound message
including receipts, an empty allowlist blocks everything, and
`base/auth_email.py` is the only module that calls `message.send()`.

New this batch: `python manage.py email_safety_check [--to addr]` reports what
a deployment would do — whether it can send, whether the allowlist is in
force, and whether a given address would be reached — **without sending
anything and without printing any credential**. It exists because the safe
default is only reassuring if someone can confirm it is in force *before* the
SMTP credentials go in.

`ses-email-setup.md` §4 now specifies the order the variables must be applied
in: **allowlist first, credentials last**, with the check run in between.
Setting the credentials before the allowlist is the one mistake in that runbook
that cannot be undone, because the mail has already left.

**Blockers, all owner actions:** verify an SES sender domain with DKIM, create
separate QA and production SMTP credentials, set the QA variables in the stated
order, and request production access for the production identity only.
Applying Railway variables is a deploy, so it waits for the deploy gate.

### Maps

**The exposed key has no live consumer anywhere this project controls.**
Neither app depends on Google Maps (no `google_maps_flutter` in either
`pubspec.yaml` or lockfile; both render with `flutter_map` over OSM), and no
Railway service has a `GOOGLE_MAPS_API_KEY` variable — not `backend`, not
`celery`, not `web`, and the production backend service has no variables at
all. The key is in no file at HEAD of any repo; it survives in two rider
commits, which is enough to require rotation.

Revoking it should therefore break nothing. Two things still need the owner:

1. **Confirm no out-of-band consumer** — a different Cloud project, a script,
   a spreadsheet add-on. The Cloud Console usage graph for the key answers
   this in one page. Live usage and current restrictions **cannot be checked
   from here**, and the key was deliberately not probed with a test request:
   that would send a production credential to a third party and put a line in
   the owner's billing to learn what the console states directly.
2. **QA needs a key it has never had.** All four `/ride/maps/*` proxies return
   **HTTP 502 `API_ERROR`** when the key is unset, so on QA today: destination
   search returns nothing, a chosen suggestion cannot be resolved, the pickup
   pin shows no address, and there is no route line or ETA. The **rider** app
   calls all four; the **driver** app calls none.

   This is a device-certification blocker for the rider, and it reorders the
   plan: there is no QA rotation to perform, because QA never had a key.
   Create a *new* restricted server key for QA (restricted to those four APIs,
   with a daily quota cap and a billing alert), and revoke the exposed one
   independently.

Also still committed: the Firebase Android API key in the rider's
`google-services.json`. That is how Firebase config is meant to ship, so it is
not the same class of leak, but it needs an Android application restriction
(package + SHA-1) or it can be spent from anywhere.

---

## 7. Remaining P0 items, and the exact steps to Android device certification

### P0, in dependency order

| # | Item | Owner | Blocks |
|---|---|---|---|
| 1 | Back up both keystores + `credentials.txt` off-machine, reachable by a second person | Owner | every signed artifact |
| 2 | Decide the driver `applicationId` (recommendation: `com.saaradhigo.driver`) | Owner | the first Play upload, permanently |
| 3 | Create a restricted **QA** Maps server key | Owner | rider destination search, route and ETA on QA |
| 4 | Configure SES for QA, allowlist first | Owner | email login, password reset, receipt delivery |
| 5 | Confirm no out-of-band consumer of the exposed Maps key, then revoke | Owner | closing the exposure |
| 6 | Decide whether driver FCM is in pilot scope | Owner | ride offers to a backgrounded driver app |
| 7 | Decide what to do about `dev` being permanently red (missing EC2 secrets) | Owner | trust in CI |

### Exact steps for Android device certification

Steps 1–4 are preconditions; 5 onward is the pass itself.

1. **Clear the keystore gate.** Back up both `.jks` files and
   `credentials.txt`, then in each app:
   `export SAARADHIGO_KEYSTORE_BACKUP_VERIFIED=yes`.
2. **Create `android/key.properties`** in each app from `credentials.txt`
   (both gitignored), or set the four `ANDROID_*` variables.
3. **Set the QA Maps key** and the QA SES variables (allowlist first), then
   run `python manage.py email_safety_check --to <QA address>` against QA and
   confirm it reports the allowlist in force.
4. **Confirm QA migrations** — 7 already applied; check for new ones.
5. **Build the QA artifacts:** `tool/build_release.sh qa-apk` in each app.
   Install the **arm64-v8a** split on at least one **Android 14+** device.
6. **Confirm the artifact is what you think it is:** the script prints the
   signer for each split APK — it must show your key, not `CN=Android Debug` —
   and the app must show the QA banner (driver) or environment badge (rider)
   on device.
7. **Run `android-device-test-checklist.md`** start to finish. It was updated
   this batch with the QA Maps and email limitations, the new build commands,
   the banner expectation, the session-end checks that only a device can prove
   (logout then force-stop; logout offline; reinstall over the top; re-enrol
   biometrics while signed in), and the store-artifact checks.
8. **Prove the upgrade path:** `adb install -r` a second signed build over the
   first. A signature mismatch here is how a pilot ends with drivers unable to
   take an update.
9. **Complete one full trip end to end** on real devices — dispatch, accept,
   navigate, complete, fare, receipt — and confirm the receipt email arrives
   with its PDF.
10. **Record every UNKNOWN that remains**, rather than inferring a pass.
    Device-dependent items stay UNKNOWN until a device says otherwise.

### Still UNKNOWN

No Android device or emulator is available in this environment. Nothing in
this batch or the last has run on a phone: the Android 14 foreground-location
behaviour, sustained GPS through screen-off, the signed install and upgrade
path, email and receipt delivery, logout with a degraded Keystore, driver
onboarding and KYC upload, and one full trip remain unverified on hardware.
iOS remains entirely unaddressed — the rider's bundle id is still the template
`com.example.vahango`, which cannot be submitted.

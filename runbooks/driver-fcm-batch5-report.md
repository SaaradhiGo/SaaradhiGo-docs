# Driver FCM, application ID, and CI presentation — batch 5

**Date:** 2026-10-17

**Not done, per the constraints:** nothing merged to `dev` or `main`, no
Railway deployment or infrastructure change, no live payments, no production
customer notification, no Play upload, no secret committed or printed, and no
signed release artifact.

---

## 1. Changes and commits

| Repo | Branch | Commits this batch |
|---|---|---|
| `SaaradhiGo-driver` | `feat/driver-dev-hardening` | `99ccd5e`, `5f43b70` |
| `SaaradhiGo-backend` | `feat/android-pilot-batch1` | `a31bb0c` |
| `SaaradhiGo-docs` | `docs/india-launch-readiness` | this report + the QA integrations checklist |

### `99ccd5e` — the application ID is settled

`com.saaradhigo.driver`, with `namespace` moved to match and
`MainActivity.kt` relocated to `kotlin/com/saaradhigo/driver/` (Kotlin
requires the directory to match the package, and the manifest resolves
`.MainActivity` against the namespace).

Two placeholders went with it, because both are things a driver reads: the
launcher label was `saaradhigo_driver` — the Flutter default, lowercase with
an underscore, shown under the icon — and is now "SaaradhiGo Driver"; the iOS
bundle id and display name were aligned so the ids do not have to be decided
twice.

`test/application_id_test.dart` (6 tests) pins the four places that must
agree. Getting the Kotlin package or directory wrong fails the build loudly;
getting `applicationId` alone wrong builds, installs, runs, and creates the
Play listing under a name nobody intended — permanently. That is the case
worth a test.

### `5f43b70` — real FCM in the driver app

See §3.

### `a31bb0c` — three backend findings

Push configurability, logout token cleanup, and the CI presentation fix. See
§3 and §2.

---

## 2. CI results

| Repo | Jobs | Result |
|---|---|---|
| `SaaradhiGo-driver` | analyze-and-test, build-android, release-config | ✅ green |
| `SaaradhiGo-backend` | validate / lint, validate / test suite, validate / postgres concurrency | ✅ green; `deploy to EC2 (optional)` skipped |

Local, before pushing: driver `flutter analyze` clean and **411 tests pass**
(362 before, 49 new); `flutter build apk --debug` succeeds. Backend `ruff`
clean and **1,320 tests pass** (1,308 before, 12 new), with the same 9
pre-existing Redis-dependent failures.

### The CI presentation fix (approved item 7)

`dev` had been red for four consecutive runs. In every one, lint, test and
postgres-concurrency **passed** and only the EC2 deploy failed, at "Setup
SSH", because `EC2_SSH_KEY` / `EC2_HOST` are not configured. A workflow's
conclusion is the worst of its jobs, so the branch showed a red X meaning "the
code is broken" when it meant "EC2 is not set up". A permanently red default
branch teaches people to stop reading CI, which is how a real failure gets
ignored.

The three gates are now named `validate / …` and the deploy job `deploy to
EC2 (optional)`. A preflight step draws the distinction:

| Secrets | Behaviour |
|---|---|
| **absent** | the job **skips**, with a `::notice` and an explanation in the run summary. Nothing is deployed, and the run is green — because nothing about the application is wrong. |
| **present** | every step runs, and a failure is **red**, exactly as before. A deploy that is configured and broken is a real failure. |

**No security gate was weakened.** `needs: [test, lint,
postgres-concurrency]` is unchanged, so an unvalidated commit still cannot
reach EC2; the branch and event conditions (`push` on `refs/heads/dev`) are
untouched; and nothing was made `continue-on-error`.

---

## 3. Driver FCM implementation evidence

### What was there

`'device_token': 'dummy_device_token'`, sent at every login and stored
verbatim. Every driver row held a token that could never receive anything,
while dispatch's own filter — `if getattr(r.user_id, 'fcm_token', None)` —
counted those drivers as push-capable and queued an FCM message for each.

Sending nothing is strictly better than sending a placeholder: the login view
only writes `fcm_token` when one is supplied, so a login from a device that
cannot produce a token leaves a previously good one alone. The key is now
**omitted** — never a placeholder, never a null.

### Three backend findings, which explain why none of this could have worked

1. **Nothing could configure Firebase, anywhere.**
   `servers/auth_user/services.py` looked only for a service-account *file* at
   `FIREBASE_CREDENTIALS_PATH` — a setting that did not exist in
   `base/settings.py`, so the lookup always fell back to
   `firebase-credentials.json` in `BASE_DIR`. That file is gitignored and is
   not in the deployed image, so `firebase_admin` was never initialised and
   `send_push_notification_task` returned `False` at its `_apps` check before
   it ever looked at a token.

   **So push has never worked in any environment, for either app**, however
   correct the clients were. Dispatch's FCM leg has been dead and the
   WebSocket has carried every ride offer alone. `FIREBASE_CREDENTIALS_JSON`
   now holds the service-account JSON itself, which is what a platform with
   environment variables and an ephemeral filesystem can supply.

2. **Logout left the device token on the row.** The refresh token was
   blacklisted but `fcm_token` was not cleared, so dispatch kept addressing
   ride offers to a handset whose session had ended — and to whoever held that
   phone next. Now cleared with a narrow write, and deliberately not fatal.

3. **`GET /ride/trip/<id>/` refuses a driver who is not yet assigned**, which
   is what makes the architecture below necessary rather than chosen.

### The architecture, and why

The plugin lives behind a `PushTransport` interface. `firebase_messaging`
throws `MissingPluginException` in a test binding and `Firebase.initializeApp()`
needs a real `google-services.json`, so a direct implementation would have
been untestable end to end. `FirebasePushTransport` is thin glue; everything
that could hide a defect sits above the seam and is tested with a fake.

| Piece | Responsibility |
|---|---|
| `PushOffer` | Parses the payload as dispatch really sends it |
| `PushRegistrar` | Permission, the login token, rotation, logout deletion |
| `RideOfferCoordinator` | The single arbiter for both channels |

**A push announces; the socket describes.** The push carries `{trip_id, type}`
and the trip-detail endpoint refuses an un-assigned driver, so there is no API
that will tell them who the rider is or where. A push therefore brings the app
forward and makes sure the dispatch socket is connected; the socket's own
`ride_request` frame supplies the details and is the only thing that can
present the sheet. If no description arrives within 12 seconds, the driver is
told the ride is gone rather than left looking at a dashboard.

*Recorded as a backend gap:* dispatch could carry the offer fields in the push
`data`, or expose an offers endpoint. Either would let a tapped notification
present the ride immediately. Not changed here — it alters a dispatch payload
two apps consume.

**`trip_id` arrives as a String over push and an int over the socket**, because
`send_push_notification_task` stringifies every data value. Both parse. That
difference is invisible until a driver taps a notification and nothing happens.

**Duplicate and stale protection**

| Rule | Why |
|---|---|
| A trip presented once is never presented again | Dispatch runs up to three waves; a driver in range of more than one is offered the same trip again |
| A settled trip stays settled for the session | Stops a wave-two push re-raising the sheet over the ride screen and inviting a second accept |
| Offers older than 90s are dropped silently | The backend's own `TRIP_ACCEPT_TIMEOUT_SECONDS`; after it `auto_cancel_trip` has closed the trip regardless |
| Nothing is presented while a trip is active | The driver could not take it anyway |
| Settling happens *before* the accept call | A second announcement can land while the accept is in flight |

The authoritative protection against double acceptance remains server-side
driver exclusivity, which is already enforced and tested. These rules stop the
app from *offering* what the server would refuse.

**The terminated-app path.** `getInitialMessage()` hands the launch message
over once and it must be read before the first frame, so `main()` starts the
stack against the same `ProviderContainer` the widget tree uses. A broadcast
stream drops events nobody is listening to yet, so the coordinator retains the
offer until the dashboard exists to show it.

**Notification channel** at `Importance.max` — a ride offer has seconds of
value — plus a foreground heads-up notification, because Android displays
nothing by itself for a message that arrives in the foreground: a driver with
the app open and the screen off would have seen nothing.

**The WebSocket path is unchanged.** It is still the only channel that presents
an offer, and the app works exactly as before on a build with no Firebase
configuration at all.

### Gradle

The `google-services` plugin is applied **only when
`android/app/google-services.json` exists**. The plugin fails the build when
the file is missing, so applying it unconditionally would mean nobody without
the file — including CI, which needs no push to run tests — could build,
analyse or test the app.

Verified both ways: the build succeeds with the file **absent** (which is
itself the proof that the plugin was not applied, since applying it without
the file is a hard failure) and with a syntactically valid one **present** for
`com.saaradhigo.driver`. The fake config used for that check was deleted
immediately and the path is gitignored.

Core library desugaring was enabled because `flutter_local_notifications`
requires it — without it the build fails at `:app:checkDebugAarMetadata`.

### Tests — 49 new in the driver app, 12 in the backend

| File | Covers |
|---|---|
| `push_offer_test.dart` (11) | the stringified trip id, the int form, whitespace, other notification types, unusable ids, hostile payloads, tap provenance, age, and clock skew not reading as fresh |
| `ride_offer_coordinator_test.dart` (22) | socket presents; never twice; nothing while on a trip; a push alone does not present; the socket frame that follows is marked as expected; expiry when it never comes; staleness at 91s and 8 hours; no send time is not assumed stale; a settled trip cannot be re-offered by either channel; the foreground, launch-notification and background-tap wiring |
| `push_registration_test.dart` (16) | token after permission; declined permission and unconfigured builds yield null without asking; a hostile transport cannot take the app down; rotation PATCHed as `fcm_token`; failures swallowed; login sends the real token, **omits** the key when there is none, never sends the placeholder, and cannot be failed by a throwing reader; logout deletes the local token even when the server call fails and even when the deletion throws |
| `test_push_registration_lifecycle.py` (12) | logout clears the token, still succeeds when the clear fails, never echoes it; a login with no `device_token` does not wipe a stored one; a rotation replaces it; write-only and absent from the profile response; the JSON env var preferred over the file and passed as a parsed dict; unusable JSON reported without raising or logging the key; a missing file is not an error; an unconfigured deployment skips the push cleanly |

The logout test fails against the previous view. **Physical delivery is
UNKNOWN** and stays so until a device verifies it: the tests cover every path
above the FCM seam with a fake transport and cannot prove Google delivered
anything.

---

## 4. QA Maps and SES configuration checklist

Written up in full in **`qa-external-integrations-checklist.md`**, with the
exact API restrictions, per-API quota figures, billing-alert guidance, the
Railway variables, and the cheap verification command for each. Summary:

- **Maps:** a **server** key (not an Android key — the backend makes these
  calls so the apps never hold one), restricted to exactly five APIs (Places,
  Place Details, Geocoding, Directions, Distance Matrix), with daily and
  per-minute caps and a budget alert. An IP restriction will break QA until
  Railway static egress is configured, so leave it at *None* and rely on the
  API restriction plus quotas. Verification is five curl requests, one per
  API — not a loop, not a load test.
- **SES:** apply in order — `AUTH_EMAIL_ALLOWLIST` **first**, SMTP credentials
  **last**, with `manage.py email_safety_check` run in between and again
  after. Setting credentials before the allowlist is the only step that cannot
  be undone. Both `backend` and `celery` need the variables; receipts are sent
  from a Celery task.
- **Firebase:** add an Android app for `com.saaradhigo.driver` to the existing
  project, take its `google-services.json`, and set
  `FIREBASE_CREDENTIALS_JSON` on both services. This turns push on for the
  rider at the same time.

**Validation (approved item 10) has not been run**, because the configuration
does not exist yet. Each section states the exact cost-controlled command to
run once it does.

---

## 5. Signing backup verification status

**NOT VERIFIED.** Stated plainly rather than guessed at.

What was actually checked: both keystores and `credentials.txt` are present
and readable with their stored passwords and expected aliases, valid to 2054;
there is **no second copy** anywhere under the user profile; the directory is
deliberately **outside** every sync root, so there is no incidental cloud
copy; and there is no encrypted archive of them on this machine.

An off-machine encrypted backup — a password-manager entry or a vault — **is
not observable from this machine at all**, so no amount of local checking can
confirm one. The `SAARADHIGO_KEYSTORE_BACKUP_VERIFIED` gate in both build
scripts is an **assertion by whoever sets it**, not a verification by the
tooling. Nothing in this batch set it, and **no signed artifact was produced**.

The checklist gives a restore-and-verify procedure that makes the backup
provable rather than asserted: restore from the vault to a scratch directory,
`keytool -list` it with the backed-up password, and compare the SHA-1 to the
known value. A match proves the backup is the same key *and* that the
backed-up password opens it — the mistyped-password failure is not detectable
any other way.

---

## 6. Remaining Android device-testing blockers

Everything here is **UNKNOWN**. No Android device or emulator exists in this
environment.

| # | Blocker | Needs first |
|---|---|---|
| 1 | Keystore backup verified | owner |
| 2 | Signed APK/AAB install, and upgrade over install | #1 |
| 3 | **Push delivered foregrounded** | Firebase service account + `google-services.json` |
| 4 | **Push delivered backgrounded** | same |
| 5 | **Terminated-app notification tap navigates to the offer** | same |
| 6 | **No duplicate sheet when push and socket both arrive** | same — the logic is tested, the race is not |
| 7 | Android 14+ foreground location while on duty | device |
| 8 | Sustained GPS through screen-off | device, hours |
| 9 | Rider destination search, route, ETA | QA Maps key |
| 10 | Email login, password reset, receipt with PDF | SES |
| 11 | Logout, including with a degraded Keystore | device; re-enrol biometrics while signed in |
| 12 | Driver onboarding and KYC upload end to end | device |
| 13 | One full trip: dispatch → accept → navigate → complete → receipt | all of the above |

Items 3–6 are new this batch and are the ones the FCM work cannot prove by
itself. The automated tests exercise every path above the FCM seam; whether
Google delivers to a specific handset, in a specific app state, under a
specific OEM's battery management, is a device question.

---

## 7. Exact controlled QA deployment proposal

Nothing here has been done. It is a proposal, in order, with a stop condition
at each step.

### Preconditions (no deployment yet)

1. **Verify the keystore backup** by the restore-and-compare procedure, on a
   machine that is not this one, for both keystores. Then set
   `SAARADHIGO_KEYSTORE_BACKUP_VERIFIED=yes`.
2. **Create the QA Maps server key** with the restrictions and quotas in the
   checklist.
3. **Create the Firebase Android app** for `com.saaradhigo.driver` and the
   service account.
4. **Create the QA SES identity** and SMTP credentials.

> Stop if any of these cannot be completed. Each one blocks a different part
> of the acceptance pass, and partial configuration produces a pass that
> cannot be interpreted.

### The deployment itself — one service, one variable set, in order

5. **Approve a QA deployment window.** Applying Railway variables restarts a
   service; this is the first approved deployment in several batches, and it
   should be deliberate rather than folded into other work.
6. **Apply QA variables to `backend` and `celery`, in this order:**
   `AUTH_EMAIL_ALLOWLIST` → confirm `ENVIRONMENT` → `GOOGLE_MAPS_API_KEY` →
   `FIREBASE_CREDENTIALS_JSON` → the `EMAIL_*` block → `EMAIL_HOST_USER` and
   `EMAIL_HOST_PASSWORD` last.
7. **Do not merge anything to `dev`.** The feature branches are what gets
   deployed, by pointing the QA service at a branch for the window, or by a
   controlled merge that is a separate decision. Railway's watch patterns stay
   as they are; nothing should auto-deploy.
8. **Verify, in this order, before any device work:**
   - `manage.py email_safety_check --to <QA> --to <non-QA>` — reports the
     allowlist in force and the non-QA address refused.
   - `manage.py shell -c "...firebase_admin._apps"` — reports initialised.
   - five curl requests, one per Maps proxy — 200 with results.
   - the backend `/healthz` check Railway already runs.

> Stop if any verification fails. A device pass against a half-configured QA
> produces UNKNOWNs that look like failures.

### Then the device pass

9. **Build the QA artifacts** with `tool/build_release.sh qa-apk` in each app.
   Install the **arm64-v8a** split on at least one Android 14+ device, and
   confirm on the device that the driver app shows the red `QA` banner and the
   rider app its environment badge. An artifact whose environment you infer
   from the command you typed is not verified.
10. **Run `android-device-test-checklist.md`** start to finish, covering
    blockers 2–13, and record every UNKNOWN that remains rather than inferring
    a pass.
11. **Report back before anything is merged or published.** Live payments stay
    disabled and nothing goes to the Play Console.

### What this proposal deliberately does not do

- No merge to `dev` or `main`, and no change to Railway's watch patterns.
- No production environment changes; the production service remains as it is,
  with its staged deletion still staged.
- No Play Console account, listing or upload.
- No live payment enablement.
- No real customer email or SMS: the allowlist is applied before any
  credential, and QA stays in the SES sandbox.

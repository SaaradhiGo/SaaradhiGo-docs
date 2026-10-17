# Autonomous batch 6 — push reliability, contract verification, launch gaps, hardening

**Date:** 2026-10-17
**Mode:** autonomous, six-hour window, no owner interaction.

**Branches are committed locally and NOT pushed.** This batch's brief
authorised committing but not pushing, so **no CI has run on any of it** —
every CI result below is from batch 5 or earlier and is marked accordingly.
Nothing was merged, deployed, published or sent.

---

## 1. Changes and commits by repository

### `SaaradhiGo-driver` — `feat/driver-dev-hardening` (6 commits)

| Commit | What |
|---|---|
| `83d9714` | Ride-lifecycle contract conformance (15 tests) |
| `be887ce` | Rate the passenger on a completed trip (10 tests) |
| `d163fc3`, `3667171` | Analyzer clean-ups from the above |
| `dfd290b` | Log redaction by key substring, not an exact list (9 tests) |
| `b265a9a` | Take the offer expiry from the server instead of guessing (4 tests) |

### `SaaradhiGo-mobile` — `feat/android-pilot-batch1` (8 commits)

| Commit | What |
|---|---|
| `54c9fe9` | Register rotated push tokens; stop sending `'dummy_token'` (13 tests) |
| `3217521` | "No notifications" stops meaning "the fetch failed" (8 tests) |
| `f303670` | Logout ends the session on the server, not just locally (5 tests) |
| `6bc197f` | Ride-lifecycle contract conformance (15 tests) |
| `31fb86b` | Help & Support stops being decorative — real tickets (16 tests) |
| `0433933` | A completed trip can be rated (14 tests) |
| `3a33acf` | A payment session was being written to the log (13 tests) |
| `b5161db` | A cancellation that never reached the server (10 tests) |

### `SaaradhiGo-backend` — `feat/android-pilot-batch1` (3 commits)

| Commit | What |
|---|---|
| `9fca4df` | Ride lifecycle pinned to a generated contract — 17 envelopes (18 tests) |
| `7087c1d` | Object-level authorization probed, not just role gates (28 tests) |
| `00e1eff` | Offer push carries an expiry, and nothing else (6 tests) |

### `SaaradhiGo-docs` — `docs/india-launch-readiness`

This report and `android-release-and-incident-runbook.md`.

---

## 2. Defects discovered and fixed

Nine, all with a test that fails against the previous code.

### P0 — a rider could not cancel a ride

`POST /ride/trip/<id>/cancel/` **requires** `reason`, validated against
`{no_show, wrong_location, too_long, safety, other}`. Every call site invoked
`cancelTrip(token, tripId)` with no reason, so the body was `{}` and **every
rider cancellation was refused** with `INVALID_REASON`. The 400 was caught and
`debugPrint`ed, and the app cleared its local state regardless — so the rider
saw "cancelled" while the trip stayed live, a driver could still be dispatched
to them, and the trip only ended when `auto_cancel_trip` ran 90 seconds later.

Found indirectly: an authorization test asserting only "the request was
refused" passed for the wrong reason. Chasing *why* it was refused surfaced
this.

### P0 — push had never worked anywhere, and could not be configured

`servers/auth_user/services.py` looked only for a service-account **file** at
`FIREBASE_CREDENTIALS_PATH`, a setting that did not exist in
`base/settings.py`. The file is gitignored and not in the deployed image, so
`firebase_admin` was never initialised in any environment and every push was
skipped at the `_apps` check — for **both** apps, for the project's life.
(Fixed in batch 5; restated because it gates every push row in §5.)

### P1 — a payment session was written to the log

`ride_service.dart` called `debugPrint(response.body)` on the response from
`POST /payments/create-order/`. That body carries the Cashfree
**`payment_session_id`** — the credential the checkout SDK opens a payment
with — so every order creation put a payment session into logcat, readable by
any app with log access on a rooted or shared handset. Two further call sites
interpolated `${response.body}` for payment and wallet-top-up errors.

### P1 — a push token was written to the log

The driver's `LoggingInterceptor` logs request bodies with key-based
redaction, matched **exactly**. The list had `device_token` but not
`fcm_token` — so the moment the app began PATCHing `{'fcm_token': ...}` to
register a rotated token, that body was logged in full. A device token is a
credential: anyone holding one can push to that handset. Both apps now match
by key **substring**, which is what the backend's own filter does and says why:
exact enumeration "would inevitably miss one".

### P1 — the rider's logout was local only

`clearSession()` was `TokenStore.clear()` alone. The refresh token stayed
valid for its full 14-day life, so a copy taken off a lost or sold phone kept
minting access tokens after the rider pressed Log out; and `POST
/auth/logout/` — which is what clears `fcm_token` server-side — was never
called, so dispatch kept addressing push to that handset and to whoever held
it next.

### P1 — rotated push tokens were never registered (rider)

`PushNotificationService` listened to `onTokenRefresh` and only
`debugPrint`ed the new value. FCM replaces a token on reinstall, on device
restore and after a long idle period, so the server kept a dead token until
the rider's next login and every push in between went nowhere.

### P1 — `'dummy_token'` at rider login

`otp_verification_screen.dart` defaulted `deviceToken` to the literal
`'dummy_token'` and only overwrote it when FCM produced something. Every login
from a device without push stored that literal, and dispatch's filter
(`if getattr(r.user_id, 'fcm_token', None)`) counted the rider as
push-capable. The key is now omitted, which preserves whatever good token the
server already had.

### P1 — "no notifications" meant "the fetch failed"

`NotificationService` returned `{notifications: [], hasNext: false}` for
**every** non-200, the provider had no error state, and the screen rendered
"No notifications yet" whenever the list was empty. An expired access token
produced this reliably — tokens live 15 minutes and nothing on that path
refreshed them — so a rider who opened notifications a quarter of an hour into
a session was told they had none. The provider also used an implicit downcast
that threw on an unexpected shape and was swallowed.

### P2 — the offer expiry was a client-side guess

The push payload carried no expiry, so the driver app compared the FCM send
time against a hard-coded 90 seconds copied from
`TRIP_ACCEPT_TIMEOUT_SECONDS`. A guess about server behaviour living in a
client, wrong in both directions the moment the timeout is retuned. The server
now sends `expires_at`; the heuristic remains as a fallback.

### Recorded, not a defect

The two cancel endpoints have **different** valid-reason vocabularies — the
driver's accepts `vehicle_issue` and `personal`, the rider's accepts
`wrong_location` and `too_long`. Nothing in the backend relates them, and
sending one app's set to the other's endpoint is a 400. Both apps now pin
their own.

---

## 3. Feature coverage changes

### Rider — three capabilities that did not exist

| Capability | Before | Now |
|---|---|---|
| **Support tickets** | Help & Support rendered six tiles, **none with a handler** | A real form: eight server-defined issue types, optional trip attachment, a reference number, four honest states, the server's own error text, and the description surviving a failure |
| **Rating a trip** | No way to submit one; `RideStatus.rated` unreachable | Stars + optional comment from the payment-pending banner, skippable, each star separately labelled for screen readers |
| **Notification error state** | Failure looked identical to an empty inbox | Distinct error state with Try again; 401 refreshed and retried |

"Live Chat / Usually responds in minutes" was removed: there is no chat
service behind the app, so it promised something that does not exist. The
support address was `support@saaradhigo.com`; the platform runs on
saaradhigo.**in**, so the .com address was a dead end.

### Driver — one capability, one correctness change

| Capability | Before | Now |
|---|---|---|
| **Rating the passenger** | None | Stars + comment from the trip summary, refusals kept apart |
| **Offer staleness** | Hard-coded 90s guess | Server's `expires_at`, heuristic as fallback |

### Both — contract conformance

17 ride-lifecycle envelopes captured from real views and asserted in both
apps. The sharpest assertion in each: **every trip status the server can send
is handled**. The rider must map all six (an unmapped `reached` previously
gave a rider with a driver outside a blank screen); the driver must decode
each to its **own** value, because its generated decoder ends in
`?? TripStatus.requested` and would otherwise present an in-progress trip as
brand new.

---

## 4. Tests: passed, failed, skipped

Local, this machine, after the final commit.

| Suite | Result |
|---|---|
| Backend `ruff check .` | ✅ clean |
| Backend `pytest` | **1,372 passed**, 45 skipped, **9 failed**, 102 deselected |
| Driver `flutter analyze` | ✅ clean |
| Driver `flutter test` | **449 passed** |
| Driver Android config gate | ✅ 21 passed |
| Driver `flutter build apk --debug` | ✅ builds |
| Rider `flutter analyze` | ✅ no errors (40 pre-existing infos) |
| Rider `flutter test` | **249 passed**, 7 skipped |
| Rider Android config gate | ✅ 13 passed |
| Rider `flutter build apk --release` | ✅ **refused** by the signing guard, as designed |

Net new this batch: **+139 backend**, **+23 driver**, **+68 rider**.

### The 9 backend failures are pre-existing

7 in `test_durable_dispatch.py`, 2 in `test_executive_revenue.py`. All
Redis-dependent; confirmed identical on `HEAD` without this batch's changes in
batch 3 and unchanged since. **No local Redis** (`127.0.0.1:6379` refuses).
CI has one and they pass there.

### The 7 rider skips are pre-existing

`skip: true` markers in `personal_info_screen_test.dart` (4) and
`profile_tab_test.dart` (3) — rider profile UI paths no test covers.

### UNKNOWN, not passed

| Check | Why |
|---|---|
| **PostgreSQL concurrency tests** (102 deselected) | No local PostgreSQL (`127.0.0.1:5432` refuses). `SELECT FOR UPDATE` is a no-op on SQLite, so running them here would be worse than not: they would pass and mean nothing. CI runs them with a service container; green in batch 5. |
| **Redis-dependent tests** | No local Redis. |
| **CI for this batch** | Not pushed, so not run. |
| **Everything physical** | No device or emulator. See §6. |

---

## 5. Verified versus UNKNOWN integration behaviour

### Verified locally, against real server output

- Authentication, token refresh and logout, including a rotated refresh pair
  driven through the real `AuthInterceptor`.
- Object-level authorization: 28 refusals hold — another rider or an
  unassigned driver cannot read, cancel or rate a trip; the start OTP never
  reaches a third party; earnings, history and support tickets are scoped to
  the caller; six admin surfaces are closed to riders and drivers; anonymous
  callers refused everywhere.
- Fare quote shape, including `currency`, `validated_km` and `pricing_source`,
  and the 400 envelope for an unserved area.
- Trip record parsing by both apps, with string-serialised Decimal
  coordinates becoming doubles.
- Rating: success, and all four refusals kept apart.
- Rider and driver history pagination; earnings gross/net/commission;
  withdrawal gating fields; incentives at the doubled path; elite status.
- SOS returning a reference.
- Push offer payload: `expires_at` derived from the real setting, and a test
  feeding a full offer in to assert the rider's name, addresses, coordinates
  and fare are **absent** from what a lock screen would show.
- Receipt delivery gated by `AUTH_EMAIL_ALLOWLIST`, empty allowlist blocking
  everything.

### UNKNOWN — needs owner configuration

| Integration | State |
|---|---|
| **FCM delivery** | No Firebase service account in any environment, no `google-services.json` for the driver. Every path above the FCM seam is tested with a fake transport; nothing proves Google delivered anything. |
| **Google Maps** | QA has no `GOOGLE_MAPS_API_KEY`; all four `/ride/maps/*` proxies return **502**. Rider destination search, pickup address, route and ETA are dead on QA. |
| **SES email** | QA has no email variables. Email verification, password reset and receipt delivery cannot be exercised. |
| **Cashfree** | Sandbox only; live payments disabled and untouched. |

---

## 6. Remaining P0 / P1

### P0 — blocks a controlled QA release

| # | Item | Owner? |
|---|---|---|
| 1 | **Keystore backup verified** off-machine by the restore-and-compare procedure | Owner |
| 2 | **Firebase**: Android app for `com.saaradhigo.driver` + `FIREBASE_CREDENTIALS_JSON` on both services | Owner |
| 3 | **QA Maps server key**, restricted to five APIs with quotas and a budget alert | Owner |
| 4 | **QA SES**: allowlist first, credentials last | Owner |
| 5 | **Push delivery on a device**, all three app states | Needs 2 |
| 6 | **One full trip on two real phones** | Needs 1–4 |

### P1

| # | Item |
|---|---|
| 7 | `dev` CI is red only because the EC2 deploy secrets are absent (presentation fixed in batch 5; the secrets decision is open) |
| 8 | `cancelMayNotHaveReachedServer` is set but not surfaced in the rider UI — the root cause is fixed, the warning path is not wired to a screen |
| 9 | Rider profile screens have 7 skipped tests and no coverage |
| 10 | A tapped push cannot present a full offer: `GET /ride/trip/<id>/` refuses an un-assigned driver, so the socket must describe it. Dispatch could carry more offer fields, but anything richer than `expires_at`/`vehicle_type` is lock-screen-visible and was deliberately left out |
| 11 | iOS entirely: the rider's bundle id is still `com.example.vahango` |
| 12 | 40 analyzer infos in the rider app and 7 unformatted files in the driver, both pre-existing |

---

## 7. Owner-controlled cloud configuration

Unchanged from batch 5 and fully specified in
`qa-external-integrations-checklist.md`. Nothing in this batch could advance
them: they need consoles and credentials this environment does not have.

1. Keystore backup → then `SAARADHIGO_KEYSTORE_BACKUP_VERIFIED=yes`.
2. Firebase Android app + service account → the only thing that makes §5's
   push rows testable.
3. QA Maps **server** key (not an Android key — the backend makes those calls).
4. QA SES identity and SMTP credentials, **allowlist applied first**.

---

## 8. Consolidated QA deployment and Android certification plan

1. **Clear the four preconditions above.** Stop if any cannot be completed —
   partial configuration produces a pass that cannot be interpreted.
2. **Approve pushing the four branches**, so CI runs on this batch for the
   first time. Expect: driver 449 + rider 249 Flutter tests, backend 1,372
   plus the 102 PostgreSQL-marked tests that only CI can run, and the 9
   Redis-dependent failures turning green on CI's Redis.
3. **One approved QA deployment window**, applying variables to `backend` and
   `celery` in this order: `AUTH_EMAIL_ALLOWLIST` → confirm `ENVIRONMENT` →
   `GOOGLE_MAPS_API_KEY` → `FIREBASE_CREDENTIALS_JSON` → the `EMAIL_*` block →
   `EMAIL_HOST_USER`/`EMAIL_HOST_PASSWORD` last. No merge to `dev`; Railway
   watch patterns unchanged.
4. **Run the four QA environment checks** in
   `android-release-and-incident-runbook.md` §3 before touching a device.
5. **Build QA artifacts** with `tool/build_release.sh qa-apk`; install the
   arm64 split; confirm the QA banner **on the device**.
6. **Run the 22-step acceptance script** (runbook §6) and the push matrix
   (§5), recording every UNKNOWN rather than inferring a pass.
7. **Prove the upgrade path** with `adb install -r`.
8. **Report before any merge or upload.** Live payments stay disabled.

---

## 9. Security-sensitive findings

Two needing attention, both fixed here, neither previously reported.

### A payment session in the device log — **fixed**

`debugPrint(response.body)` on the payment-order response wrote the Cashfree
`payment_session_id` to logcat on every order creation. Readable by any app
with log access on a rooted or shared handset. Fixed by substring-based
redaction, with a structural test that scans `lib/` for any log line
interpolating a raw response body.

**Residual exposure:** any build already distributed with this code wrote
those sessions to device logs. Live payments are disabled and QA uses the
Cashfree sandbox, so the sessions logged so far are sandbox sessions. No
production payment session can have leaked because production payments have
never been enabled. Worth confirming against the Cashfree dashboard that no
production credentials were ever configured in QA.

### A push token in the device log — **fixed**

The driver's exact-match redaction list missed `fcm_token`. Anyone holding a
device token can send a push to that handset. Fixed the same way, in both
apps.

### Boundaries confirmed intact

28 object-authorization probes all refuse correctly, including the start OTP
never reaching a third party and support tickets — which can be safety
complaints — being scoped to their author. One note on method: each test
asserts the error **code**, not just the status. The first version asserted
only `status in (401, 403, 404)` and three tests "passed" on a 400 that turned
out to be a validation refusal reached *before* the ownership check. A test
satisfied by a refusal for the wrong reason would have certified an open
boundary as closed.

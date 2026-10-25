# QA activation and device certification — handoff checklist

*Prepared 2026-10-17. Nothing in this document was executed: every step needs
a console, a credential, a phone or a deployment that this environment does
not have. No code audit was performed for it — it reads the already-verified
state and turns it into an ordered procedure.*

**Branch state (all pushed, all CI green, all trees clean):**

| Repo | Branch | Head |
|---|---|---|
| backend | `feat/android-pilot-batch1` | `cfc721a` |
| rider | `feat/android-pilot-batch1` | `88ebaeb` |
| driver | `feat/driver-dev-hardening` | `8b6060d` |
| docs | `docs/india-launch-readiness` | `5ea72b2` |

> **Two corrections landed after this checklist was first written
> (2026-10-17).** The rider's cash button now calls the server and verifies
> the result before navigating, and the `0006_email_identity` migration now
> pre-checks over-long addresses. Both are reflected in gate 5, §2.1 and
> steps 16–16c below. Everything else here remains unexecuted: it still
> needs consoles, credentials, a deployment or a phone.

**Verification run for those two corrections (2026-10-17):**

| Suite | Result |
|---|---|
| Backend, default runner | **1,441 passed**, 9 skipped (was 1,417 + the 24 new tests) |
| Backend, `-m postgres` | **106 of 107 passed** against PostgreSQL 15 in Docker |
| Rider `flutter test` | **280 passed**, 7 pre-existing skips |
| Rider `flutter analyze` | 0 errors; the 40 pre-existing infos, unchanged |

The one PostgreSQL failure is
`test_lifecycle_under_gps_load.py::test_the_driver_does_not_receive_its_own_location_echo`.
It is **pre-existing and order-dependent, not a regression**: it passes on its
own, it passes alongside the new migration tests, and it fails the same way
with the new test file excluded entirely. CI is the authority on that suite --
it reported all 102 postgres tests green before this batch. Do not read it as
a signal about these corrections.

---

## 0. What the word "verified" means here

Read this before anything else, because the rest of the document uses these
three labels and they are not interchangeable.

| Label | What was actually done | What it cannot tell you |
|---|---|---|
| **SOFTWARE-VERIFIED** | Code read, and a test asserts it. 459 driver tests, 259 rider, 1,417 backend in CI, 9/9 jobs green | Nothing about Google, AWS, Cashfree or a handset. Every external edge is a fake in these tests |
| **LIVE-SERVICE UNKNOWN** | The code path is software-verified, but the third-party side has never been configured | Whether the key, credential or quota works |
| **DEVICE UNKNOWN** | No physical Android device has run either app at any point in this work | Push delivery, GPS behaviour, background survival, the actual UI |

Three things are **SOFTWARE-VERIFIED and simultaneously DEVICE UNKNOWN**, and
they are the ones most likely to be mistaken for done:

- **Push.** Every path above the FCM seam is tested with a fake transport.
  That Google delivers a message to a handset in foreground, background and
  terminated-tap states is unproven.
- **Background GPS.** The manifest, the `foregroundServiceType="location"`
  declaration and the permission flow are tested. That Android does not kill
  the service on a given OEM build is unproven.
- **Signing.** The build scripts refuse to sign without the backup
  assertion. That the keystores survive the loss of one laptop is unproven —
  see gate 1.

---

## 1. Five owner gates, in order

Gates 1–4 are the four known external dependencies. Gate 5 was found while
preparing this handoff and is new.

Each gate states: **do**, **verify**, **expect**, **stop if**, **roll back by**.

### Gate 1 — Keystore backup *(blocks every signed artifact)*

**Do.** Copy both keystores *and* `credentials.txt` from
`C:\Users\srini\SaaradhiGo-signing\` into a vault a second person can reach.
Then, **on a different machine**, restore them to a scratch directory.

**Verify.**

```bash
keytool -list -v -keystore saaradhigo-driver-upload.jks -alias saaradhigo-driver-upload
```

**Expect.** SHA-1 matches exactly:

- Driver `94:1B:99:B0:32:51:CF:09:92:A3:AE:C3:C3:B5:8A:FD:60:27:51:0F`
- Rider  `71:3A:45:6A:B2:C0:C4:FF:FF:43:FF:A6:E5:66:30:37:A2:D9:2B:1D`

A match proves the backup is the same key **and** that the stored password
opens it — a password mistyped into the vault is the failure mode this
catches, and nothing else catches it. Repeat for both keystores, then delete
the scratch copy.

**Stop if.** The SHA-1 differs, or the password does not open the restored
copy. Do not set the environment variable; do not build a signed artifact.

**Roll back by.** Deleting the scratch restore. Nothing else changed — this
procedure only ever copies, never moves, the original keystores.

Only after a match: `export SAARADHIGO_KEYSTORE_BACKUP_VERIFIED=yes`. That
variable is an assertion by whoever sets it, not a verification by the
tooling.

### Gate 2 — Firebase *(blocks all push, both apps)*

**Do.**

1. Firebase console, project `saaradhigo-4f8ad`: add an **Android app** with package `com.saaradhigo.driver`.
2. Download `google-services.json` into `SaaradhiGo-driver/android/app/`. It is gitignored, and the Gradle plugin applies only when the file is present.
3. Google Cloud console: create a **service account** with the Firebase Messaging role, download its JSON key.
4. Railway: set `FIREBASE_CREDENTIALS_JSON` to that JSON on QA **`backend`** *and* **`celery`**.

**Verify.**

```bash
# local, after step 2, in SaaradhiGo-driver
flutter build apk --debug

# server, after step 4 and the deploy in section 2
python manage.py shell -c "from servers.auth_user import services; print(bool(services.firebase_admin._apps))"
```

**Expect.** `google-services.json found — FCM is enabled for this build.`,
then `True`.

**Stop if.** `False` after the variable is set — the JSON is malformed or
truncated. Do not paste it into a ticket or a log to inspect it.

**Roll back by.** Deleting the Railway variable and redeploying. Push returns
to being skipped at the `_apps` check, which is today's behaviour, not a new
failure. Deleting the Firebase Android app is not required and not advised; an
unused registration is harmless.

> Step 4 turns push on for the **rider** at the same time — it is already
> registered as `com.deeptrics.saaradhigo`.

### Gate 3 — Google Maps server key *(blocks rider search, route, ETA)*

**Do.** Create a **server** key, not an Android key: these calls are made by
the Django backend, which is the point of the proxies — the apps never hold a
Maps key. API restriction: exactly Places, Geocoding, Directions, Distance
Matrix. Application restriction **None** for now, because Railway gives a
service no stable outbound IP without static egress. Daily caps per
`qa-external-integrations-checklist.md` §2, plus a budget alert — Distance
Matrix is called on every fare quote and is the largest runaway-cost risk in
the platform. Set `GOOGLE_MAPS_API_KEY` on QA **`backend`** only; `celery`
does not call these routes.

**Verify.** Five requests total, one per API:

```bash
curl -s -X POST "$QA_API/ride/maps/geocode" -H "Authorization: Bearer $QA_TOKEN" \
     -H 'Content-Type: application/json' -d '{"input":"Koramangala"}' | head -c 300
```

then `place-details`, `reverse-geocode`, `directions`.

**Expect.** HTTP 200 with results. A **502 `API_ERROR`** means the key is
unset or that API is not enabled; a 403 in the backend log means the
restriction is too tight.

**Stop if.** Any 403 — widen the restriction rather than removing it. Do not
loop these calls; five requests answer the question.

**Roll back by.** Deleting the variable and redeploying: the four proxies
return 502 and the rider falls back to today's behaviour. The key can be
deleted in the Cloud Console independently.

> The **exposed** production key is a separate matter, untouched by this gate:
> it has no live consumer anywhere this project controls. Confirm that from
> the Cloud Console usage graph, then revoke it. See `maps-key-rotation.md`.

### Gate 4 — SES email *(blocks email login, password reset, receipts)*

**Do.** AWS side first: verify the sender domain with DKIM, create **QA-only**
SMTP credentials, add SPF/DMARC and a bounce destination, and leave QA in the
SES sandbox. Then set the Railway variables on **both** `backend` and
`celery`, in this order:

1. `AUTH_EMAIL_ALLOWLIST` = `@deeptrics.com`
2. confirm `ENVIRONMENT` is **not** `production`
3. `EMAIL_BACKEND` = `django.core.mail.backends.smtp.EmailBackend`
4. `EMAIL_HOST` = `email-smtp.ap-south-1.amazonaws.com`
5. `EMAIL_PORT` = `587`, `EMAIL_USE_TLS` = `True`
6. `DEFAULT_FROM_EMAIL` = `no-reply@qa.saaradhigo.com`
7. `EMAIL_HOST_USER` / `EMAIL_HOST_PASSWORD` — **last**

The order *is* the safety. Credentials before the allowlist opens a window in
which the deployment can email anybody, and that is the one step here that
cannot be undone, because the mail has already left.

**Verify.** After step 6, and again after step 7:

```bash
python manage.py email_safety_check --to qa.lead@deeptrics.com --to someone@gmail.com
```

It sends nothing and prints no credential.

**Expect.** After 6: cannot send, no password. After 7: can send, **and** the
non-allowlisted address is refused.

**Stop if.** It reports it can send while the allowlist is empty or absent.
Remove `EMAIL_HOST_PASSWORD` immediately and re-check.

**Roll back by.** Deleting `EMAIL_HOST_USER` / `EMAIL_HOST_PASSWORD` first —
that alone restores "can email nobody" — then the rest. Keep
`AUTH_EMAIL_ALLOWLIST`; it is the thing you want left behind.

> An empty allowlist reports *"allowed to email nobody"*. That is the safe
> default, not a misconfiguration. Receipts pass through the same gate, so a
> receipt for a non-allowlisted rider is blocked on QA. Expect that rather
> than debugging it.

### Gate 5 — Payments: not activatable, and that is the correct state *(new)*

QA has `CASHFREE_ENVIRONMENT` and `CASHFREE_WEBHOOK_SECRET` but **not**
`CASHFREE_APP_ID` or `CASHFREE_SECRET_KEY`. `get_available_gateways()`
therefore returns an empty list, and no online payment session can be
created.

**Do: nothing.** The acceptance test below settles in **cash**, which needs no
gateway. Enabling even sandbox Cashfree carries cost and live-payment
adjacency, and is not required to certify the ride lifecycle.

**Cash now settles properly (corrected 2026-10-17).** The two defects that
made the cash path cosmetic are fixed, so the acceptance test below exercises
a real settlement rather than a screen change:

- The rider's *"Pay with Cash instead?"* button used to be
  `context.push('/rate-driver')` and nothing else — no API call at all. It now
  calls `POST /payments/switch/`, reads the trip back, and only then decides
  where the rider goes. A refusal or an unanswered request keeps them on the
  payment screen with the fare still showing.
- The server's cash branch wrote the replacement payment as
  `method='wallet'`, which the driver's cash-confirmation handler could not
  find — so the driver got *"No cash payment found for this trip"* and the
  fare could never settle. It now writes `method='cash', status='pending'`.

**What to expect on the phone:** choosing cash is *not* payment. The trip
carries a **pending** cash payment until the **driver** confirms collection,
which is a separate step in the test below. A rider who is shown the rating
screen after a *failed* cash switch is a regression, not a quirk.

**Stop if.** Anyone proposes putting production Cashfree credentials on QA.

---

## 2. One consolidated QA deployment

### 2.1 The migration — safe and reversible, with one unguarded failure mode

The release carries **exactly one** new migration:
`auth_user/0006_email_identity.py`.

| Check | Result |
|---|---|
| Model changes with no migration | **none** — `makemigrations --check` reports "No changes detected" |
| Other apps' migrations vs `dev` | **none** — one file in the whole diff |
| Reverse defined | **yes** — the `RunPython` has an explicit no-op reverse |
| Reverse actually executed | **yes, on PostgreSQL 15** — `tests/test_email_identity_migration.py::TestOnPostgres` migrates 0006 → 0005 → 0006 and asserts the column width and the unique index against the database each way |

What it does: lower-cases existing addresses, adds a nullable
`email_verified_at`, narrows `email` from `varchar(256)` to `varchar(254)`,
and adds a case-insensitive partial unique index.

**Two ways it can fail — both now guarded (corrected 2026-10-17):**

1. **Duplicate addresses.** The migration refuses with a `RuntimeError`
   naming the count and the account ids but never the addresses, so a
   deployment log does not become a list of customer emails. Which account
   keeps an address is a support decision — one of them may hold a wallet
   balance — so it is deliberately not automatic.
2. **An address longer than 254 characters.** This was unguarded and is the
   second correction in this batch. A stored 255- or 256-character address is
   legal in the column as it stands, and the narrowing failed with a raw
   `value too long for type character varying(254)` inside
   `preDeployCommand`. Confirmed against PostgreSQL 15 rather than assumed.
   The pre-check now reports it with the count, the longest length and the
   account ids — and reports it *together* with any duplicates, so one run
   tells you everything to fix.

The pre-check also now **checks first and writes after**: normalisation used
to run before the duplicate check, so a refused migration had already
modified data.

The two read-only checks below are therefore no longer load-bearing — the
migration will refuse safely on its own. They are still worth two minutes,
because seeing the problem before the deploy window is cheaper than seeing it
inside one:

```sql
SELECT lower(email) AS e, count(*) FROM auth_user_customuser
 WHERE email IS NOT NULL AND email <> '' GROUP BY 1 HAVING count(*) > 1;

SELECT count(*) FROM auth_user_customuser WHERE length(email) > 254;
```

**Expect.** Zero rows, and `0`.

**Stop if.** Either returns anything. Clear or shorten the offending addresses
first. If you skip these checks and deploy anyway, the migration refuses with
a message naming the account ids and the deployment aborts cleanly — that is
now a safe failure rather than a raw database error.

**Why a failure here is survivable anyway:** QA `backend` runs
`preDeployCommand: python manage.py migrate --noinput`. A failed pre-deploy
aborts the release and the **current deployment keeps serving** — an
outage-free failure, not a half-migrated database.

**Roll back by.**

```bash
python manage.py migrate auth_user 0005
```

That drops `email_verified_at` (losing only verification timestamps, which do
not exist yet), removes the unique index, and widens the column. Lower-cased
addresses stay lower-cased, which is not a loss: `email__iexact` lookups
already treated them as equal.

### 2.2 Minimum Railway change

Confirmed read-only today, QA environment `a2863403`:

| Service | Source branch | Autodeploy | Migrates |
|---|---|---|---|
| `backend` | `dev` | suppressed (`watchPatterns: .railway-autodeploy-disabled/**`, `checkSuites: false`) | yes, via `preDeployCommand` |
| `celery` | `dev` | suppressed, same pattern | no |

**The minimum change is one field on two services:** point `backend` and
`celery` — QA only — at `feat/android-pilot-batch1`. Do **not** merge to `dev`
to achieve this.

Because autodeploy is suppressed, changing the branch deploys nothing by
itself; a manual deploy is still required. That property is load-bearing —
leave the watch patterns alone.

### 2.3 Sequence

1. Run the two SQL pre-checks (§2.1). Stop on any row.
2. Set the gate 2/3/4 variables on `backend` and `celery`. **Allowlist before credentials.**
3. Point both QA services at `feat/android-pilot-batch1`.
4. Deploy **`backend` first and alone** — it is the one that migrates.
5. Watch `/healthz` and the pre-deploy log. **Expect** `Applying auth_user.0006_email_identity... OK` and a healthy check inside the 600 s timeout.
6. Deploy `celery` only after `backend` is healthy.
7. Run the server-side verifications, no phone needed: the gate 2 `firebase_admin._apps` check, the gate 3 five curls, the gate 4 `email_safety_check`.

**Stop if.** `/healthz` does not go healthy, or the pre-deploy log shows the
`RuntimeError`. Do not retry the deploy — fix the data.

**Roll back by.** Re-pointing both services at `dev` and redeploying the last
known-good deployment, **`4228caf` of 2026-10-16 08:55**, which is still the
latest Railway deployment today. If 0006 had already applied, reverse it per
§2.1 *before* the old image starts, because the old code does not know the
new column.

---

## 3. QA build commands

Both scripts refuse to run without `SAARADHIGO_KEYSTORE_BACKUP_VERIFIED=yes`
(gate 1) and without signing material present.

```bash
cd SaaradhiGo-mobile && ./tool/build_release.sh qa-apk    # rider
cd SaaradhiGo-driver && ./tool/build_release.sh qa-apk    # driver
```

Each produces split-per-ABI release APKs in `build/app/outputs/flutter-apk/`
with `ALLOW_NON_PRODUCTION_HOST=true` and the QA banner. Install the
`arm64-v8a` APK.

**Verified configuration — do not "fix" the asymmetry below:**

| | Rider | Driver |
|---|---|---|
| applicationId | `com.deeptrics.saaradhigo` *(unchanged — never change a shipped id)* | `com.saaradhigo.driver` |
| dart-define | `BASE_URL` / `WS_BASE_URL` | `API_BASE_URL` / `WS_BASE_URL` |
| QA REST | `https://backend-qa-811d.up.railway.app/api/v1` | `https://backend-qa-811d.up.railway.app/api/v1/` |
| QA WS | `wss://backend-qa-811d.up.railway.app/ws` | `wss://backend-qa-811d.up.railway.app/` |

The two apps use **opposite** WebSocket conventions: the rider's base carries
`/ws` and its paths do not; the driver's paths carry `ws/` and its base must
not. Copying one app's value into the other produces `/wsws/...`. Both are
pinned by `base_url_composition_test.dart`, so a cross-contaminated value
fails CI rather than shipping.

**Expect** both builds to be reproducible from the heads in the table at the
top. **Stop if** either script prints the backup refusal — that means gate 1
is not done, not that the script is broken.

---

## 4. Two-phone acceptance test

**Prerequisites:** gates 1–4 complete, §2 deployed, two physical Android
phones (one rider, one driver), both on mobile data, QA APKs sideloaded.

**Log in with numbers that are in `TEST_PHONE_NUMBERS`** — already set on QA.
Those log in with a fixed OTP and **send no SMS**. Do not use an arbitrary
real number: that sends a real SMS, which the engagement rules exclude.

Two safety pre-checks before starting:

- **Clear `emergency_contact` on the QA rider account, or set it to a tester's own phone.** `dispatch_sos` sends a real SMS to that number through SNS, and the AWS credentials on QA are live. This is the one step in the test that can reach an uninvolved person.
- Confirm the test riders' email addresses are inside `AUTH_EMAIL_ALLOWLIST`, or the receipt will be correctly blocked and will look like a bug.

| # | Step | Expect | Stop if |
|---|---|---|---|
| 1 | Both apps launch | QA banner visible on both | No banner — a production build was installed |
| 2 | Rider and driver log in (test numbers) | Home screen, no 404s | Any 404 — that build's REST base URL is wrong |
| 3 | Driver goes online | Status persists; the location permission prompt explains itself | A permission dialog with no rationale |
| 4 | Driver app **backgrounded** 10 min | Location still updating server-side; foreground notification visible | Service killed — record the OEM and Android version |
| 5 | Rider searches a destination | Autocomplete returns results | 502 `API_ERROR` → gate 3 incomplete |
| 6 | Rider books | Fare quote shown; driver receives the offer | — |
| 7 | **Push, foreground** — driver app open | Offer appears immediately | — |
| 8 | **Push, backgrounded** — driver app in background | System notification; tapping it opens the offer | — |
| 9 | **Push, terminated** — force-stop the driver app, book again | Notification arrives; the tap **launches** the app onto the offer | Any of 7–9 failing → gate 2, or the device's battery optimisation |
| 10 | Two drivers accept the same offer | Exactly one wins; the other is told it is gone | Both accept — stop the test and report |
| 11 | **Rider cancels** before pickup | The trip clears **only after** the server confirms | The screen clears while offline — the ordering defect is back |
| 12 | Cancel with the phone in airplane mode | The trip **stays on screen**; the rider is told it may not have reached the server | The trip disappears |
| 13 | Re-book, driver arrives, starts the trip | Live tracking on both sides | — |
| 14 | **SOS** during the trip | Ops push; SOS recorded | An SMS reaches anyone unexpected → the pre-check above was skipped |
| 15 | Driver completes the trip | Rider sees the payment screen with the correct fare | — |
| 16 | **Payment** — rider taps *"Pay with Cash instead?"* | A message naming the amount to hand over, then the rating screen. Server-side the trip is now `payment_method: cash`, `payment_status: pending` | The rating screen opens with no message, or after an error — the switch is not being verified |
| 16a | Repeat step 16 with the phone in airplane mode | The rider **stays** on the payment screen and is told the trip has not been paid | They are moved on — a failed switch is being treated as a payment |
| 16b | **Driver confirms cash collection** | The trip moves to `payment_status: completed`; the driver's earnings reflect it once | *"No cash payment found for this trip"* — the `method='cash'` fix is not in the deployed build |
| 16c | Driver confirms a second time | Idempotent: reported as already confirmed, and GMV is **not** double-counted | A second confirmation adds earnings again |
| 17 | Rider rates the driver | Accepted once; a second attempt is settled, not an error | — |
| 18 | **Receipt** | Email with PDF arrives for the allowlisted rider within ~2 min (Celery) | Nothing after 5 min → check the `celery` log, then gate 4 |
| 19 | **KYC** — driver uploads a document | Upload succeeds to S3; status moves to pending review | — |
| 20 | Rider raises a support ticket | It persists and is visible to admin | — |
| 21 | Both log out | Server session revoked, push registration cleared; relaunching does not restore the session | A logged-out session returns |

Steps 4 and 7–9 are the **DEVICE UNKNOWN** items from §0 and are the reason
this test exists. Everything else in it is already software-verified and is
here to catch an integration or configuration mistake, not a logic defect.

Steps 16–16c are the exception worth naming: the cash path was corrected on
2026-10-17 and its tests are **SOFTWARE-VERIFIED** (21 rider tests, 12
backend), but no device has walked it end to end, and the driver-side
confirmation in 16b crosses the trip WebSocket, which no unit test exercises.

---

## 5. Stop conditions that apply throughout

- No merge to `dev` or `main`.
- No production deployment, and no change to the `production` environment or the `SaaradhiGo-backend` service.
- No signed artifact before gate 1 matches, and no bypassing `SAARADHIGO_KEYSTORE_BACKUP_VERIFIED`.
- No production payment credentials anywhere, QA included.
- No real customer SMS, email or push. Test numbers and the allowlist are how that is enforced — do not widen either to make a step pass.
- No secret pasted into a ticket, a log or a chat in order to "check" it.
- Do not remove the Railway watch patterns; suppressed autodeploy is a safety property.
- Do not set Cashfree credentials to "make the payment step pass". The
  acceptance test settles in cash by design; online settlement stays
  unexercised on QA.
- Stop if a rider is ever moved off the payment screen by a cash switch that
  did not succeed, or if a driver cannot confirm collection for a trip the
  rider switched to cash. Both are the defects corrected on 2026-10-17
  reappearing, and both mean a fare that silently never settles.

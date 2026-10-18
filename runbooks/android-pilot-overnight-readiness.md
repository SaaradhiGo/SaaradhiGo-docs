# Android pilot — overnight readiness, 2026-10-18

Written for: the owner/CTO, and whoever runs the next controlled QA
deployment and the two-phone certification.

This batch deliberately spent most of its effort **exercising the deployed QA
system** rather than adding tests. Test counts are reported, but every claim
below that matters is backed by something that actually ran against
`cfc721a` on Railway QA.

---

## 0. The one-line state

The admin console, the rider API surface and the driver API surface were
driven end to end against deployed QA and **work**. One real security gap was
found and fixed (a password change did not end existing sessions). One
documentation defect was found and fixed (the Maps verification command used
a field the backend does not read, so it would have failed even with a
correct key). QA MFA enforcement is **DEFERRED BY OWNER**. Everything still
blocking the pilot is external: signing-key backup, Firebase, Maps key, SES.

---

## 1. Commits and verification

| Repo | Branch | Head | Change |
|---|---|---|---|
| backend | `feat/android-pilot-batch1` | `aed14a7` | session revocation on credential change |
| rider | `feat/android-pilot-batch1` | `88ebaeb` | unchanged this batch |
| driver | `feat/driver-dev-hardening` | `8b6060d` | unchanged this batch |
| docs | `docs/india-launch-readiness` | *(this commit)* | Maps field fix + this page |

| Suite | Result |
|---|---|
| Backend, default runner | **1,454 passed**, 9 skipped |
| Backend, `-m postgres` | **107 passed in CI** on `aed14a7` (27 + 80). See the note below |
| Backend `ruff` | clean |
| Rider `flutter analyze` / `flutter test` | 0 errors (40 pre-existing infos) / **280 passed**, 7 skipped |
| Driver `flutter analyze` / `flutter test` | **No issues found** / **459 passed** |
| **Functional verification against deployed QA** | **20 PASS, 0 FAIL, 1 UNKNOWN** |

The functional run is the one to read. It is in §3.

**CI on `aed14a7`: all four jobs green** — `validate / test suite` 1,454
passed, `validate / postgres concurrency` **107 passed**, `lint, SAST,
dependency audit` clean, EC2 deploy skipped.

**The local PostgreSQL picture, measured rather than assumed.** A first local
run reported 5 failures and 17 errors, on a laptop simultaneously running two
Flutter suites against a freshly started container. Re-run on an idle machine
with the container ready, it reproduces CI exactly: **1 failed, 106 passed,
107 collected**. (The first run's counts exceeded the collected total because
17 were *teardown* errors on tests whose call phase had passed — database
contention, not broken behaviour.)

The one remaining local failure is
`test_lifecycle_under_gps_load::test_the_driver_does_not_receive_its_own_location_echo`.
It is **pre-existing and order-dependent**: it passes on its own, it passes
in CI, and it fails the same way with this batch's new test files excluded
entirely. It is not a regression and not caused by any change here.

---

## 2. Defects fixed, with evidence

### 2.1 A password change did not end existing sessions *(security, fixed)*

`change_password` and `reset_password` set the new password and stopped. A
comment said revocation was handled — true of *reset* tokens, whose generator
hashes the password hash, and **not** true of the JWTs the apps and the
console actually carry.

A refresh token lives **14 days** and `ROTATE_REFRESH_TOKENS` reissues one on
every use. So the single action a person takes when they believe their
password is compromised left the attacker's session alive and self-renewing.
On an operator account — which approves KYC and releases payouts — that is
the difference between a contained incident and an uncontained one.

Fixed in `servers/auth_user/session_revocation.py`: every outstanding refresh
token is blacklisted through the `token_blacklist` app that was already
installed and already used by logout. An operator's credential change also
writes an `AdminAuditLog` row; a rider's does not, so routine traffic cannot
bury KYC approvals in the audit log.

**Evidence:** 13 tests, which assert a pre-change refresh token being refused
by the real `/api/v1/auth/refresh/` endpoint rather than by reading the
blacklist table. Also asserted: every session goes, not only the caller's; a
*failed* attempt revokes nothing; the reset flow behaves the same.

**The deliberate limit, recorded so it is not mistaken for an oversight:** an
access token already minted stays valid for its **15 minutes**. Killing one
mid-flight would mean a database read on every authenticated request. A test
asserts that window still exists, so changing it later is a decision rather
than an accident.

### 2.2 The Maps verification command could not succeed *(documentation, fixed)*

Every runbook told the owner to prove the Maps key with:

```
-d '{"search_text":"Koramangala"}'
```

`proxy_geocode` reads **`input`**, not `search_text`. That request returns
`400 MISSING_FIELDS` regardless of the key — so the check would have failed
with a perfectly good key and sent someone hunting a phantom Google problem.

Confirmed against deployed QA and corrected in three documents. The exact
required field for all four proxies is now tabulated in
`qa-external-integrations-checklist.md` §2:

| Route | Required JSON |
|---|---|
| `maps/geocode` | `{"input": "..."}` |
| `maps/place-details` | `{"place_id": "..."}` |
| `maps/reverse-geocode` | `{"lat": …, "lng": …}` |
| `maps/directions` | `{"origin_lat": …, "origin_lng": …, "dest_lat": …, "dest_lng": …}` |

**The apps were already right.** The rider's `location_service.dart` sends
exactly these names, verified field by field. Only the documentation was
wrong — which is the kind of defect that costs an evening rather than a
release.

---

## 3. What was proven against deployed QA

Run in-process inside the QA `backend` container against `cfc721a`. A
temporary operator and rider were created with a random password, used, and
deleted; the owner's real admin account was never authenticated as, modified
or locked out.

### The admin console genuinely works

| Check | Result |
|---|---|
| Operator password sign-in through the real form | **PASS** — HTTP 302 → `/`, session authenticated |
| **All 16 console pages render** | **PASS** — 16/16 HTTP 200, none redirected, none broken, no page rendering a traceback |
| Detail pages against real rows (`ride_detail`, `driver_profile`) | **PASS** — both 200, read-only |
| Anonymous visitor | **PASS** — 302 to `/login/` |
| A rider with correct credentials | **PASS** — authenticates but **cannot reach the dashboard** |

Pages verified: fleet monitor, driver onboarding, dispute support, payment
dashboard, executive revenue, driver loyalty, rides, riders, promo codes,
emergency, stale rides, transactions, fare surge, predictive heatmaps,
notifications, global search.

### Security controls genuinely hold

| Check | Result |
|---|---|
| Brute-force lockout on the console | **PASS** — locked after **8** failed attempts; the *correct* password is also refused while locked |
| Operator cannot authenticate through the SMS/OTP path | **PASS** — no token minted (one SIM swap must not yield an operator token) |
| Operator API login does not enumerate accounts | **PASS** — unknown phone and wrong password both return an identical `401 AUTH_INVALID_CREDENTIALS` |
| Rider token on operator endpoints | **PASS** — 3/3 refused |
| **Production MFA cannot be switched off** | **PASS** — with `IS_PRODUCTION=True` and `OPS_MFA_ENFORCED=False`, `mfa_enforced()` is still `True` |
| MFA infrastructure preserved | **PASS** — all policy functions present, `RECOVERY_CODE_COUNT=10` |
| Maps proxies with no key | **PASS** — **4/4** return a clean `502 API_ERROR`, logging `GOOGLE_MAPS_API_KEY is not set.` — not a 500, not a fake success |
| Empty SES allowlist | **PASS** — blocks every probe address; `send_message` refuses with `blocked: … AUTH_EMAIL_ALLOWLIST is empty` |
| QA has no working SMTP | **PASS** — `EMAIL_HOST is unset or localhost` |
| Password change revokes sessions | **UNKNOWN** — the fix is newer than the deployed commit; verified locally by 13 tests, pending the next QA deployment |

### Earlier in the day, on the same deployment

27 PASS / 0 FAIL covering OTP auth, email/password auth, token refresh,
logout, driver KYC and vehicle endpoints, rider fare quote, rider
cancellation, **cash switch**, **driver cash confirmation over the real trip
WebSocket**, confirmation idempotency, SOS, and receipt generation with a PDF.

After every run QA was returned to its exact baseline: 3 users, 51 trips, 28
payments, 3 SOS events, 0 test rows.

---

## 4. MFA — DEFERRED BY OWNER

**Why `mfa_required_for(staff)` was false, answered:** it is by design, not a
defect. `mfa_enforced()` returns `IS_PRODUCTION or OPS_MFA_ENFORCED`. QA is
neither, so it falls through to `has_device(user)` — and the QA admin has no
enrolled device. The module calls this out as deliberate bootstrap
behaviour: "a fresh QA deployment can bootstrap an operator and enrol them
rather than locking itself out before anyone has a device."

Per the owner's decision this batch: **no mandatory MFA, no TOTP enrollment
flow, no recovery-code workflow was built.** Existing MFA infrastructure is
untouched and verified present. Production enforcement was tested and
**cannot be turned off by any variable**.

Status: **DEFERRED BY OWNER** — not a failed test, and not a completed
control.

What *was* strengthened instead, as instructed: session revocation on
credential change (§2.1), plus verification that rate limiting, role
boundaries and audit logging already work (§3).

---

## 5. Mobile readiness

No mobile code changed this batch. Both apps are green (driver 459 tests,
analyze clean; rider 280 tests, 0 analyze errors) and the launch-critical
paths listed for this phase were already closed in earlier batches:
authentication and session recovery, push token registration and refresh, GPS
and WebSocket reconnection, cancellation with server-confirmation ordering,
cash collection, SOS, support, ratings, receipts, and driver onboarding.

Two mobile-relevant facts were *verified* rather than assumed this batch:

- The rider app's four Maps payloads match the backend field-for-field.
- The cash path the rider app drives (`/payments/switch/` → driver WebSocket
  confirmation) settles a real fare on QA.

Everything that needs a handset remains **DEVICE UNKNOWN**: push delivery in
three app states, background GPS survival on a given OEM build, and the
signed-APK install/upgrade.

---

## 6. Backup, rollback and operational safety

### Current deployment state

| Item | Value |
|---|---|
| `backend` deployment | `a45cf055-301a-427e-9943-e275af4a2c4c` |
| `celery` deployment | `ba52d9e1-af3b-4ed9-ba2a-8813415bc39a` |
| Pinned commit | `cfc721ad98d33dd33f4b17baed345bb1256527c4` |
| Source | `SaaradhiGo/SaaradhiGo-backend` @ `feat/android-pilot-batch1`, **pinned by commit SHA** |
| Migration state | `auth_user` at `0006_email_identity`; **0 unapplied**; `email` is `varchar(254)`; `uniq_user_email_ci` present |
| Auto-deploy | suppressed — `watchPatterns: [".railway-autodeploy-disabled/**"]`, `checkSuites: false`; **and** the commit pin means branch pushes cannot deploy |

### Pre-deployment backup evidence

`pg_dump -Fc` → `/var/lib/postgresql/data/backups/pre-0006-20261018T021013Z.dump`,
290,417 bytes, SHA-256 `30c86ab5ac753d65703ac62144a1bf3dfb5198977779f5a5d3dd88b637973682`.
**Verified by restoring it** into a scratch database in the same instance:
`pg_restore` exit 0, zero stderr, row counts matching (users 3, trips 51,
payments 28, migrations 118). Scratch database dropped.

### The limitation, stated plainly

That dump lives on **the same Postgres volume as the database**. It protects
against a bad migration or a bad deploy. It does **not** protect against loss
of the volume. Railway's own backup APIs returned `Not Authorized` to the
CLI session, so whether platform volume backups are enabled could not be
observed from here — that is a dashboard check only the owner can make.

### Can an encrypted off-platform backup be prepared safely?

Investigated, not performed. Three options, in the order I would choose them:

1. **Railway volume backups (preferred).** Data never leaves the platform, so
   nothing is exported and no approval question arises. Enable in the
   dashboard on the `Postgres` service. This is the right answer for QA.
2. **Encrypted dump into the project's existing object store.** The project
   already stores KYC documents in its QA bucket, so an
   `openssl enc`/`gpg --symmetric` dump written there stays inside a trust
   boundary the project already uses, and survives volume loss. Needs a
   passphrase kept with the signing credentials — a backup whose key is lost
   is not a backup.
3. **Download to an operator machine.** Possible but it moves customer data
   to a new location. **Not done:** an attempt to copy the dump to this
   machine was correctly refused as PII handling.

My recommendation: option 1 now, option 2 before the pilot carries real
riders. Neither needs new cloud credentials or a billing change. **No
database data was exported.**

### Safe rollback sequence

1. Roll `backend` back to deployment `29260b08-dfa0-43fb-b19a-9648441e752c` (`4228caf` on `dev`, `canRollback: true`).
2. **Before the old image serves traffic**, reverse the migration: `python manage.py migrate auth_user 0005`. The old code does not know `email_verified_at`.
3. Roll `celery` back the same way.
4. If data is wrong rather than code: restore the verified dump into a **scratch** database first and compare, never straight over the live one.

Reversibility is proven, not assumed: `0006` was migrated `0006 → 0005 →
0006` on PostgreSQL 15 with the column width and unique index asserted each
way.

### The staged destructive patch — do not accept it

QA carries patch **`9dc1ab52-b125-4b90-952c-9a6166ece202`**, status `STAGED`,
`destructive: true`. It **deletes the `alluring-happiness` service**.

Any `accept-deploy` on the QA environment — including an incidental one from
a tool or the dashboard's "deploy staged changes" — commits that deletion.
Production carries a similar staged delete (`10532a6f-…`) for its shell
service.

This is why deployments here use the per-service manual action below rather
than committing environment changes.

### Service-specific manual deployment procedure

Repointing the source does **not** deploy, because the watch patterns
suppress the build (observed: the repoint produced a `SKIPPED` deployment
with no build). An explicit per-service trigger is required:

```graphql
mutation { serviceInstanceDeployV2(
  serviceId: "<service>", environmentId: "<env>", commitSha: "<40-hex>") }
```

Deploy `backend` **first and alone** — it is the only service with
`preDeployCommand: python manage.py migrate --noinput` — wait for `/healthz`,
then deploy `celery`. Never use `accept-deploy`.

---

## 7. Android device certification — executable procedure

Two Android 14+ handsets, QA APKs from `tool/build_release.sh qa-apk` in each
repo. **Gate: no signed artifact until the keystore backup SHA-1 match in
`qa-activation-handoff.md` gate 1 is done.** Do not bypass
`SAARADHIGO_KEYSTORE_BACKUP_VERIFIED`.

Pre-checks that prevent contacting real people:

- Clear `emergency_contact` on the test rider, or set it to a tester's own
  phone. `dispatch_sos` SMSes it through SNS and QA's AWS credentials are live.
- Log in with numbers in `TEST_PHONE_NUMBERS` — fixed OTP, **no SMS sent**.
- Confirm the test riders' addresses are in `AUTH_EMAIL_ALLOWLIST`, or
  receipts are correctly blocked and will look like a bug.

| # | Step | PASS evidence | Stop / rollback |
|---|---|---|---|
| 1 | Install signed APK, then install the next build over it | Both install; app data survives the upgrade | Install fails or data is wiped → stop; do not distribute |
| 2 | Rider and Driver log in | Home screen, QA banner visible, no 404 | Any 404 → wrong REST base URL for that build; rebuild |
| 3 | Driver KYC + vehicle onboarding | Document uploads to S3; vehicle saved; status `pending` | Upload 4xx/5xx → check presign route before blaming the device |
| 4 | Admin approves the driver in the console | Driver becomes approved; `AdminAuditLog` row exists | No audit row → stop; approvals must be attributable |
| 5 | Rider pickup + destination search | Autocomplete returns results | `502 API_ERROR` → Maps key not set (gate 3), not an app defect |
| 6 | Fare quote and booking | Quote shown; trip created | — |
| 7 | **Driver push + acceptance** — foreground, background, and force-stopped | Offer arrives in all three; tap from terminated launches onto the offer | Any state failing → gate 2 (Firebase) or the OEM's battery optimisation; record the OEM and Android build |
| 8 | **Background GPS**, driver app backgrounded 10 min | Location still updating server-side; foreground notification visible | Service killed → record OEM + Android version; this is the top device risk |
| 9 | Arrival + pickup OTP | OTP accepted, trip starts | — |
| 10 | Trip completion | Rider sees payment screen with correct fare | — |
| 11 | **Cash collection** — rider taps "Pay with Cash instead?", then driver confirms collection | Rider is told the amount to hand over; trip becomes `payment_method: cash`, `payment_status: completed` after the driver confirms; a second confirm is idempotent | Rider moved on with no message, or driver sees "No cash payment found" → the corrected cash path is not in the installed build |
| 11a | Repeat step 11 in airplane mode | Rider **stays** on the payment screen and is told the trip is not paid | They are moved on → a failed switch is being treated as payment; stop |
| 12 | Receipt | Email with PDF within ~2 min for an allowlisted rider; also retrievable from the app | Nothing after 5 min → check `celery` log, then gate 4 (SES) |
| 13 | Rider and driver ratings | Accepted once; a repeat is settled, not an error | — |
| 14 | Cancellation and reassignment | Trip clears **only after** server confirmation; a new driver can be assigned | Screen clears while offline → the ordering defect is back; stop |
| 15 | **SOS** with a designated test contact only | Ops alert recorded | An SMS reaches anyone unexpected → the pre-check was skipped; stop immediately |
| 16 | Logout on both apps | Server session revoked, push registration cleared; relaunch does not restore the session | A logged-out session returns → stop |

Steps 7, 8 and 1 are the genuinely unknown ones. Everything else is already
verified in software and is here to catch a configuration or integration
mistake.

---

## 8. Remaining owner actions

Four external dependencies, unchanged, each with its proving check in
`qa-activation-handoff.md`:

1. **Keystore backup** — restore both keystores on another machine and match the SHA-1. Blocks every signed artifact.
2. **Firebase** — Android app for `com.saaradhigo.driver` + a Messaging service account on `backend` and `celery`. Blocks all push.
3. **Maps** — a restricted QA **server** key on `backend`. Blocks rider search, route and ETA. (Verification command now correct.)
4. **SES** — allowlist first, credentials last. Blocks email login, reset and receipts.

Plus, surfaced by this work:

5. **Enable Railway volume backups** on the QA `Postgres` service (dashboard; one toggle).
6. **Decide on MFA enrollment** before production privileged access. QA is deferred; production is still gated.
7. **Do not accept** the staged QA patch `9dc1ab52-…`.

---

## 9. The exact next step

Deploy `aed14a7` to QA by the §6 procedure — `backend` first and alone, then
`celery` — so the session-revocation fix becomes live, and re-run the
functional check to turn its one UNKNOWN into a PASS. That deployment carries
**no migration**: `0006` is already applied and `aed14a7` adds none, so the
pre-deploy step is a no-op and the only rollback needed is a deployment
rollback.

Then, and only after gate 1 and gate 2 are done, build the QA APKs and run §7
on two handsets.

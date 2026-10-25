# QA external integrations — exact configuration to apply

> **The ordered procedure built from this page — with verification,
> stop conditions and rollback for each step, plus the QA deployment
> sequence and the two-phone acceptance test — is
> [`qa-activation-handoff.md`](qa-activation-handoff.md). Start there.
> This page remains the specification for *what* each integration is.**

> ## Owner action list, with the check that proves each one
> *(Re-verified 2026-10-17, batch 6. Nothing below was performed: all four
> need consoles or credentials this environment does not have.)*
>
> | # | Action | The check that proves it |
> |---|---|---|
> | **1** | Back up both keystores **and** `credentials.txt` to a vault a second person can reach | Restore to a scratch dir on another machine, `keytool -list -v`, and match the SHA-1 in §1. A match proves the backup is the same key **and** that the stored password opens it. Then `export SAARADHIGO_KEYSTORE_BACKUP_VERIFIED=yes` |
> | **2a** | Add a Firebase **Android app** for `com.saaradhigo.driver` to project `saaradhigo-4f8ad`; put its `google-services.json` in `SaaradhiGo-driver/android/app/` | `flutter build apk --debug` prints "google-services.json found — FCM is enabled for this build." The file is gitignored and the Gradle plugin applies only when present |
> | **2b** | Create a **service account** with the Firebase Messaging role; set `FIREBASE_CREDENTIALS_JSON` on `backend` **and** `celery` | `manage.py shell -c "from servers.auth_user import services; print(bool(services.firebase_admin._apps))"` → `True`. Turns push on for the rider at the same time |
> | **3** | Create a **restricted QA Maps *server* key** (not an Android key — the backend makes these calls) and set `GOOGLE_MAPS_API_KEY` on `backend` | One `curl` per proxy, five total: 200 with results. A **502 `API_ERROR`** means unset or that API not enabled; a 403 in the backend log means the restriction is too tight |
> | **4** | SES: verify the sender domain with DKIM, create QA SMTP credentials, then apply the variables **allowlist first, credentials last** | `manage.py email_safety_check --to <qa> --to <non-qa>` between steps and again after. It sends nothing and prints no credential |
>
> Restrictions and quota figures for the Maps key, and the exact SES variable
> order, are in §2 and §3 below. Do not create cloud resources, change billing
> or provision secrets on the strength of this page alone — it is the
> specification, not an authorisation.

Everything on this page is an **owner action**. None of it was applied: it
needs credentials and consoles this environment does not have, and applying
Railway variables is a deployment.

Three integrations, in the order they should be done. Each section ends with
the command that proves it worked — none of which sends anything to a real
customer.

| # | Integration | State today | Blocks |
|---|---|---|---|
| 1 | **Signing keystore backup** | **not verified** | every signed artifact |
| 2 | **Google Maps server key** | QA has none | rider destination search, route, ETA |
| 3 | **SES email** | QA has none | email login, password reset, receipts |
| 4 | **Firebase service account** | none in any environment | all push, both apps |

---

## 1. Signing keystore backup — not verified, and not verifiable from here

### What was actually checked

| Check | Result |
|---|---|
| Both keystores and `credentials.txt` present | ✅ `C:\Users\srini\SaaradhiGo-signing\` |
| Readable with their stored passwords, expected aliases | ✅ valid to 2054 |
| A second copy anywhere under the user profile | ❌ none (the only other keystore is Android's own `debug.keystore`) |
| Inside a sync root (OneDrive / Dropbox / Google Drive) | ❌ no — deliberate: private keys should not auto-upload |
| An encrypted archive of them anywhere on this machine | ❌ none |
| **An off-machine encrypted backup** | **cannot be observed from here at all** |

A password manager entry or an encrypted vault is not visible from this
machine, so no amount of local checking can confirm one exists. This is
recorded as **NOT VERIFIED** rather than guessed at either way.

The build scripts in both apps refuse to produce a signed artifact unless
`SAARADHIGO_KEYSTORE_BACKUP_VERIFIED=yes` is set. That variable is an
**assertion by whoever sets it**, not a verification by the tooling — which is
why the procedure below exists, and why nothing in this batch set it.

### How to make the backup verifiable rather than asserted

Copying a file somewhere is not a backup until it has been restored and used.
Restore it to a scratch directory and prove it is the same key with the same
password:

```bash
# 1. Restore from the backup into a scratch directory (NOT the original).
mkdir -p /tmp/ks-restore && cd /tmp/ks-restore
#    ... fetch saaradhigo-driver-upload.jks from the vault into here ...

# 2. List it with the password from the backed-up credentials.txt.
keytool -list -v -keystore saaradhigo-driver-upload.jks \
        -alias saaradhigo-driver-upload

# 3. Compare the SHA-1 with the known-good value.
#    Driver: 94:1B:99:B0:32:51:CF:09:92:A3:AE:C3:C3:B5:8A:FD:60:27:51:0F
#    Rider:  71:3A:45:6A:B2:C0:C4:FF:FF:43:FF:A6:E5:66:30:37:A2:D9:2B:1D

# 4. Remove the scratch copy.
cd / && rm -rf /tmp/ks-restore
```

A matching SHA-1 proves two things at once: the backup is byte-equivalent to
the live key, **and** the backed-up password actually opens it. A backup whose
password was mistyped into the vault is the failure mode this catches, and it
is not detectable any other way.

Do this for **both** keystores, from the vault, on a machine that is not this
one. Then set the environment variable, and only then build a signed artifact.

> Why this matters more than it sounds: with Play App Signing, a lost **upload**
> key can be reset by Google after identity checks — recoverable, slow. A lost
> **app signing** key without Play App Signing enrolment cannot be replaced at
> all, and the app can never be updated again. Either way the keys currently
> exist on one disk.

---

## 2. Google Maps — a new restricted server key for QA

QA has never had `GOOGLE_MAPS_API_KEY`. With it unset, all four proxies return
**HTTP 502 `API_ERROR`**: the wrappers in `servers/ride/maps_utils.py` return
`None` and the views turn that into an error.

### What depends on it

| Route | Google API billed | Rider-visible effect when absent |
|---|---|---|
| `POST /ride/maps/geocode` | **Places Autocomplete** | destination search returns nothing |
| `POST /ride/maps/place-details` | **Place Details** | a chosen suggestion cannot be resolved to coordinates |
| `POST /ride/maps/reverse-geocode` | **Geocoding** | the pickup pin shows no address |
| `POST /ride/maps/directions` | **Directions** | no route line and no ETA |

Fare quoting also calls **Distance Matrix**; without a key it falls back to
haversine × 1.4, which is why QA's `validated_km` differs from road distance.

The **rider** app calls all four. The **driver** app calls none — it renders
with `flutter_map` over OSM and holds no key.

### The key to create

A **server** key, not an Android key. These calls are made by the Django
backend, which is the whole point of the proxies: the apps never hold a Maps
key.

**Application restriction.** Railway does not give a service a stable outbound
IP by default, so an IP restriction will break QA unless static egress is
configured first. Until then leave the application restriction as *None* and
rely on the API restriction plus quotas — and treat the key as
medium-sensitivity, because an unrestricted key is spendable by anyone who
obtains it.

**API restriction — exactly these five, nothing else:**

- Places API *(Autocomplete + Place Details)*
- Geocoding API
- Directions API
- Distance Matrix API
- *(nothing else — not Maps SDK for Android, not Maps JavaScript API)*

**Quotas.** Set a daily cap per API. Distance Matrix is called on **every fare
quote, including every fallback probe**, which makes an uncapped key the
single largest runaway-cost risk in the platform. Suggested QA caps, sized for
a handful of test phones rather than traffic:

| API | Requests/day | Requests/minute |
|---|---|---|
| Places (Autocomplete) | 500 | 30 |
| Place Details | 500 | 30 |
| Geocoding | 500 | 30 |
| Directions | 500 | 30 |
| Distance Matrix | 1,000 | 60 |

**Billing alerts.** A budget on the Cloud project with alerts at 50% / 90% /
100% of a figure the owner is willing to lose. Quotas cap volume; a budget
alert is what tells a human.

**Separate keys per environment.** A QA key and a production key, never
shared. QA's can be revoked at any time without touching production.

### Railway variables

| Variable | Service | Value |
|---|---|---|
| `GOOGLE_MAPS_API_KEY` | `backend` (QA) | the new QA server key |

`celery` does not call these routes, so it does not need the key. Applying it
is a deployment, so it waits for the deploy gate.

### Proving it works, cheaply

```bash
# One request per API. Five calls total, well inside any free tier.
curl -s -X POST "$QA_API/ride/maps/geocode" -H "Authorization: Bearer $QA_TOKEN" \
     -H 'Content-Type: application/json' -d '{"input":"Koramangala"}' | head -c 300
```

Repeat for `place-details`, `reverse-geocode` and `directions`.

**The exact field each proxy requires** — verified against the deployed QA
backend on 2026-10-18, because the command above previously used
`search_text`, which these views do not read. That returned `400
MISSING_FIELDS` regardless of the key, so the check would have failed even
with a correctly configured key and sent you hunting the wrong problem.

| Route | Required JSON | Optional |
|---|---|---|
| `maps/geocode` | `{"input": "Koramangala"}` | `location` |
| `maps/place-details` | `{"place_id": "ChIJ..."}` | `session_token` |
| `maps/reverse-geocode` | `{"lat": 17.4450, "lng": 78.3800}` | — |
| `maps/directions` | `{"origin_lat":…, "origin_lng":…, "dest_lat":…, "dest_lng":…}` | — |

All four were confirmed on QA to return **HTTP 502 `API_ERROR`** with no key
set, logging `GOOGLE_MAPS_API_KEY is not set.` — a clean degradation, not a
500. The rider app sends exactly these field names (`location_service.dart`),
so app and backend agree; only this page was wrong.
 A 200 with
results means the key and the API restriction are right; a **502 `API_ERROR`**
means the key is unset or that API is not enabled; a 403 from Google (visible
in the backend log) means the restriction is too tight.

**Do not** loop these or drive them from a load test. Five requests answer the
question.

### The exposed key, separately

The compromised key has **no live consumer anywhere this project controls** —
no app depends on Google Maps, and no Railway service has the variable set.
Creating the QA key above does not involve it. Once the owner has confirmed
from the Cloud Console usage graph that nothing outside this project uses it,
it can be revoked independently. See `maps-key-rotation.md`.

---

## 3. SES — allowlist first, credentials last

QA has **none** of the email variables today. The order below is the safety:
setting SMTP credentials before the allowlist opens a window in which the
deployment can email anybody, and that is the one step on this page that
cannot be undone, because the mail has already left.

### Apply in this order

| Step | Variable | Value |
|---|---|---|
| **1** | `AUTH_EMAIL_ALLOWLIST` | `@deeptrics.com` (or the specific QA addresses) |
| **2** | `ENVIRONMENT` | confirm it is **not** `production` |
| 3 | `EMAIL_BACKEND` | `django.core.mail.backends.smtp.EmailBackend` |
| 4 | `EMAIL_HOST` | `email-smtp.ap-south-1.amazonaws.com` |
| 5 | `EMAIL_PORT` / `EMAIL_USE_TLS` | `587` / `True` |
| 6 | `DEFAULT_FROM_EMAIL` | `no-reply@qa.saaradhigo.com` |
| **7** | `EMAIL_HOST_USER` / `EMAIL_HOST_PASSWORD` | QA SMTP credentials — **last** |

Both the `backend` and `celery` services need these: receipts are sent from a
Celery task.

### Prove it after step 6, and again after step 7

```bash
python manage.py email_safety_check --to qa.lead@deeptrics.com --to someone@gmail.com
```

It sends nothing, prints no credential, and states whether the deployment can
send and whether each address would be reached. After step 6 it should report
that it cannot send (no password); after step 7, that it can send **and** that
the non-allowlisted address is refused.

An empty allowlist reports *"allowed to email nobody"* — the safe default, not
a misconfiguration. Receipts go through the same gate as the auth emails, so a
receipt for a rider who is not allowlisted is blocked on QA. Expect that
rather than debugging it.

### AWS-side prerequisites

Verify `saaradhigo.com` (ideally `qa.saaradhigo.com` too) with DKIM, create
**separate** SMTP credentials for QA and production, add SPF/DMARC and a
bounce destination, and request production access for the **production**
identity only. QA stays in the SES sandbox, which is a second layer on top of
the allowlist. Details in `ses-email-setup.md`.

---

## 4. Firebase service account — nothing pushes without it

New finding this batch, and it explains why push has never worked for either
app: `servers/auth_user/services.py` looked only for a service-account **file**
at `FIREBASE_CREDENTIALS_PATH`, a setting that did not exist in
`base/settings.py`. The file is gitignored and is not in the deployed image, so
`firebase_admin` was never initialised anywhere and every push was skipped at
the `_apps` check.

### What the owner needs to do

| # | Action | Where |
|---|---|---|
| 1 | Add an **Android app** to the existing Firebase project (`saaradhigo-4f8ad`) with package `com.saaradhigo.driver` | Firebase console |
| 2 | Download its `google-services.json` into `SaaradhiGo-driver/android/app/` — gitignored, and the Gradle plugin activates only when it is present | Local / CI secret |
| 3 | Create a **service account** for the project with the Firebase Messaging role, download its JSON key | Google Cloud console |
| 4 | Set `FIREBASE_CREDENTIALS_JSON` to that JSON on the `backend` **and** `celery` services | Railway |

The rider app is already registered (`com.deeptrics.saaradhigo`), so step 4
turns push on for the rider at the same time.

> The service-account key is a private key. It must never be committed,
> logged, or pasted into a ticket. The backend never logs its value — the
> error path reports the exception type only.

### Proving it works

Server side, without touching a phone:

```bash
python manage.py shell -c "
from servers.auth_user import services
print('firebase initialised:', bool(services.firebase_admin._apps))
"
```

Then, with a real device registered, trigger one dispatch on QA and read the
worker log for `FCM sent to user <id>`. A stale token is reported as
`UnregisteredError` and cleared automatically.

Physical delivery in all three app states — foreground, backgrounded, and a
tap that launches the app from terminated — is **UNKNOWN** until a device
confirms it. The automated tests cover every path above the FCM seam with a
fake transport; they cannot prove Google delivered anything.

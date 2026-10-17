# Android release, acceptance and incident runbook

One page for the people who build, certify and operate the pilot builds.
Nothing here has been executed against a store or a device: the signed-build
gate is still closed (§1) and no device exists in the build environment.

---

## 1. The gate that is still closed

**`SAARADHIGO_KEYSTORE_BACKUP_VERIFIED` is not set, and nothing in this batch
set it.** Both app build scripts refuse to produce a signed artifact until it
is. That is deliberate: the variable is an **assertion by whoever sets it**,
not a verification by the tooling.

What was checked, and what cannot be:

| Check | Result |
|---|---|
| Both keystores and `credentials.txt` present | ✅ `C:\Users\srini\SaaradhiGo-signing\` |
| Readable with their stored passwords, expected aliases | ✅ valid to 2054 |
| A second copy anywhere under the user profile | ❌ none |
| Inside a sync root (OneDrive / Dropbox / Google Drive) | ❌ no — deliberate |
| An encrypted archive of them on this machine | ❌ none |
| **An off-machine encrypted backup** | **not observable from this machine** |

### Make the backup provable, not asserted

Copying a file somewhere is not a backup until it has been restored and used.
On a machine that is **not** the build machine:

```bash
mkdir -p /tmp/ks-restore && cd /tmp/ks-restore
#   ... fetch saaradhigo-driver-upload.jks from the vault into here ...
keytool -list -v -keystore saaradhigo-driver-upload.jks \
        -alias saaradhigo-driver-upload
#   compare the SHA-1 with:
#   Driver: 94:1B:99:B0:32:51:CF:09:92:A3:AE:C3:C3:B5:8A:FD:60:27:51:0F
#   Rider:  71:3A:45:6A:B2:C0:C4:FF:FF:43:FF:A6:E5:66:30:37:A2:D9:2B:1D
cd / && rm -rf /tmp/ks-restore
```

A matching SHA-1 proves two things at once: the backup is byte-equivalent to
the live key, **and** the backed-up password actually opens it. A password
mistyped into the vault is the failure this catches and nothing else does.

Repeat for both keystores, then set the variable, then build.

---

## 2. Build commands

Both apps carry `tool/build_release.sh`. Use it rather than typing `flutter
build` by hand: a release needs the signing material, the right
`--dart-define` pair, the non-production opt-in for QA and **not** for a store
build, and the right artifact type — and any one of them wrong produces a
build that looks fine and is wrong.

| Target | Command | Produces |
|---|---|---|
| **Play (production)** | `tool/build_release.sh play-aab` | `.aab`, production URLs, upload-key signed |
| **QA acceptance** | `tool/build_release.sh qa-apk` | split-per-ABI `.apk`, QA URLs + `ALLOW_NON_PRODUCTION_HOST`, upload-key signed |
| **Pipeline smoke** | `tool/build_release.sh smoke-apk` | split `.apk`, **debug-signed**, needs no keys |

Both signed targets refuse until the backup gate above is cleared. `smoke-apk`
does not, which is how the pipeline is exercised today.

### Artifact sizes, measured

| App | Universal | arm64-v8a split |
|---|---|---|
| Driver | ~60 MB | **22 MB** |
| Rider | ~61 MB | **24 MB** |

Install the **arm64-v8a** split on any modern phone. For drivers on metered
data the difference is the whole point.

### Play needs an `.aab`

New apps have been App Bundle only since August 2021. The APK is for
sideloaded pilot distribution; the `.aab` is what the Play Console takes. The
signing guard covers `bundle` tasks as well as `assemble`, verified locally.

### Verify the artifact rather than trusting the command

The script prints the signer for every split it produces. Read it:

- `CN=Android Debug` → a smoke build. **Never distribute.**
- your own DN → a real signed build.

And confirm the environment **on the device**: the driver app shows a red `QA`
corner banner and the rider app an environment badge on every screen of a
non-production build. An artifact whose environment you infer from the command
you typed is not verified.

---

## 3. QA environment checks, before any device work

Run these in order. Each one is cheap and answers a question a device test
would otherwise answer confusingly.

```bash
# 1. Email: can it send, and is the allowlist in force? Sends nothing.
python manage.py email_safety_check --to <qa-address> --to <non-qa-address>

# 2. Push: is a Firebase service account actually loaded?
python manage.py shell -c "
from servers.auth_user import services
print('firebase initialised:', bool(services.firebase_admin._apps))"

# 3. Maps: one request per proxy. Five calls, not a loop.
curl -s -X POST "$QA_API/ride/maps/geocode" -H "Authorization: Bearer $QA_TOKEN" \
     -H 'Content-Type: application/json' -d '{"search_text":"Koramangala"}' | head -c 200
#    repeat for place-details, reverse-geocode, directions

# 4. Migrations
python manage.py showmigrations --plan | grep -c '\[ \]'   # expect 0
```

Stop if any of these fails. A device pass against a half-configured QA
produces UNKNOWNs that look like failures.

---

## 4. Android 14+ background-location acceptance

The manifest declaration is covered by a test in both apps
(`test/android_manifest_test.dart`). The **behaviour** is not testable off a
device: `startForeground()` for a location service throws
`MissingForegroundServiceTypeException` only at runtime on API 34+.

- [ ] At least one **Android 14 (API 34) or newer** device. The fix only
      matters from there.
- [ ] Driver goes on duty → the foreground-service notification appears and
      the app does **not** crash.
- [ ] Location permission flow: "While using the app" → then "Allow all the
      time". A driver who only grants foreground must be told what will stop
      working rather than silently losing dispatch.
- [ ] Screen off, app backgrounded, 30+ minutes unplugged → the trail keeps
      arriving. Record the handset: Xiaomi, Vivo, Oppo and Realme builds
      throttle this and it is a property of the phone, not a bug to fix.
- [ ] Deny background location → the app explains the consequence and does
      not crash.
- [ ] Revoke location **while on duty** → the app degrades honestly.

---

## 5. Push delivery, physical device

**Preconditions, both owner actions.** Without them every row below fails for
the same reason and the pass is meaningless:

1. A Firebase **Android app** for `com.saaradhigo.driver` and its
   `google-services.json` in `SaaradhiGo-driver/android/app/`.
2. `FIREBASE_CREDENTIALS_JSON` set on the `backend` **and** `celery` services.

Push has never worked in any environment for either app, so this is a first
run rather than a regression check.

- [ ] Driver log says push initialised, not "FCM unavailable".
- [ ] Android 13+ permission prompt appeared; a **Ride offers** channel exists
      at highest importance.
- [ ] After one QA login the driver's `fcm_token` is a real token, not null
      and not `dummy_device_token`.
- [ ] **Foreground:** offer dispatched → bottom sheet appears **once**, plus a
      heads-up notification.
- [ ] **Foreground, screen off:** the notification wakes the screen.
- [ ] **Backgrounded:** notification arrives; tapping it lands on the offer.
- [ ] **Terminated** (swiped from recents): notification arrives; tapping it
      launches the app and lands on the offer.
- [ ] **Doze:** idle and unplugged 30+ minutes, then dispatch. Record the
      result per handset.
- [ ] **Both channels, one sheet:** connected and foregrounded, so socket and
      push both arrive → the sheet appears exactly once.
- [ ] **Accept, then another wave:** no sheet reappears over the ride screen.
- [ ] **Stale notification:** leave one unopened past its `expires_at`, then
      tap → the app opens with no sheet.
- [ ] **A ride already taken:** tap its notification → "That ride is no longer
      available".
- [ ] **Logout:** push to that driver arrives nowhere, and `fcm_token` is null.
- [ ] **Reinstall then log in:** an offer still arrives (rotated token
      registered).
- [ ] **Second driver on the same handset:** dispatch to the first → the phone
      receives nothing.

---

## 6. Real-device end-to-end acceptance script

Two phones, one rider and one driver, on QA. Run in order; record every
UNKNOWN rather than inferring a pass.

| # | Step | Pass condition |
|---|---|---|
| 1 | Install both arm64 splits | Clean install, QA banner/badge visible |
| 2 | Driver OTP login | Session persists across a force-stop |
| 3 | Driver email login | Same session, wrong password shows "Invalid email or password." |
| 4 | Driver KYC upload + ops approval | Document stored; driver becomes approved |
| 5 | Driver goes on duty | Foreground-service notification; no crash on API 34+ |
| 6 | Rider search destination | Suggestions appear *(needs the QA Maps key)* |
| 7 | Rider fare quote | Amount in ₹, `validated_km` plausible |
| 8 | Rider books | Driver gets the offer over socket **and** push, one sheet |
| 9 | Driver accepts | Rider sees driver details; no second sheet |
| 10 | Driver arrives, rider gives OTP | Trip starts only with the right OTP |
| 11 | Mid-trip: kill the driver app | GPS trail resumes; trip rejoins on relaunch |
| 12 | Mid-trip: aeroplane mode 60s | Socket reconnects; no duplicate trip |
| 13 | Complete the trip | Final fare matches the quote's basis |
| 14 | Cash collection | Driver confirms; wallet/commission reflects it |
| 15 | Rider receipt | Email arrives with its PDF *(needs SES)* |
| 16 | **Rider rates the driver** | Stars submit; driver average moves |
| 17 | **Driver rates the rider** | Same, from the trip summary |
| 18 | **Rider reports a problem** | Ticket created with a reference; visible to ops |
| 19 | **Rider cancels a search** | Trip is cancelled **server-side**, not just locally |
| 20 | Rider SOS mid-trip | Ops alert and emergency contact both fire |
| 21 | Logout on both apps | Refresh token rejected afterwards; no push arrives |
| 22 | `adb install -r` a second signed build | Installs over the first, no signature error |

Steps 16–19 are new in this batch and have never run on a device.

---

## 7. Rollback and incident response

### What can and cannot be rolled back

| Layer | Rollback | How |
|---|---|---|
| Backend (Railway) | ✅ | Redeploy the previous deployment from the dashboard |
| Backend (EC2) | ✅ | `.last_good_sha` is written before every deploy |
| Database migration | ⚠️ **assume not** | Reversibility is per-migration; check before deploying |
| Sideloaded APK | ✅ | Distribute the previous APK; same signing key installs over |
| Play release | ⚠️ partial | Halt the rollout; a published version cannot be withdrawn, only superseded |
| **Signing key** | ❌ **never** | Upload key: Google can reset it. App signing key without Play App Signing: the app can never be updated again |

### If drivers stop receiving ride offers

1. Is it push or the socket? Push is the **fallback**; the socket carries
   dispatch for a foregrounded app. If foregrounded drivers are fine, it is
   push alone and not an outage.
2. `firebase_admin._apps` empty → credentials unset or invalid. Push is
   skipped and logged; dispatch continues over the socket.
3. Worker log shows `UnregisteredError` → stale tokens, cleared automatically;
   affected drivers need one more login.
4. `fcm_token` null across many drivers → something cleared them. Logout now
   clears it by design; a mass clear is not expected.

### If riders cannot book

1. `/ride/maps/geocode` returning **502** → the Maps key is unset or that API
   is not enabled. Destination search is dead; booking by map pin may still
   work.
2. `/ride/estimate-fare/` returning 400 with "outside every active zone" →
   the service zone or its rate card is missing for that area.
3. Fares look wrong but quote fine → Distance Matrix is unavailable and the
   haversine × 1.4 fallback is in use. Check `pricing_source` on the quote.

### If a rider's cancellation does not take

Fixed in this batch: the app sent no `reason`, the server refused with
`INVALID_REASON`, and the app cleared local state anyway — so the rider saw
"cancelled" while the trip stayed live. If it recurs, look for
`INVALID_REASON` in the backend log and check the reason vocabulary: the rider
endpoint accepts `{no_show, wrong_location, too_long, safety, other}`, the
driver endpoint a **different** set.

### Escalation facts worth having to hand

- Live payments are **disabled**. Any payment incident is sandbox-only.
- QA cannot email anyone outside `AUTH_EMAIL_ALLOWLIST`; an empty allowlist
  blocks everything, by design.
- `auto_cancel_trip` closes an unaccepted trip after
  `TRIP_ACCEPT_TIMEOUT_SECONDS` (90s default), which is also what the offer
  `expires_at` is derived from.
- Railway auto-deploy is suppressed on every service via watch patterns; a
  push cannot deploy.

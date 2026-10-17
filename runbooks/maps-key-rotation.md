# Google Maps API key — exposure, restrictions, rotation

Written for: the owner, who holds the Google Cloud project. Every step that
touches Google Cloud is yours; nothing here can be done from the repos.

**No key value appears in this document.** Keys are identified by a SHA-256
fingerprint so you can match them in the Cloud console without either of us
pasting a credential into a file.

---

## 1. The two keys, and which one is a problem

| Fingerprint (sha256, first 16) | What it is | Exposure | Action |
|---|---|---|---|
| `863b98dead568420` | **Google Maps API key** | **Committed.** Present since `e521d20` ("initial commit") in `SaaradhiGo-mobile`, in **two** places: `android/app/src/main/AndroidManifest.xml` and `ios/Runner/AppDelegate.swift` | **ROTATE.** Treat as compromised |
| `338d92da324c9d28` | **Firebase Android API key**, in `android/app/google-services.json` | Committed, and **this is by design** — Google ships this key inside every Android app and it is not a secret. It is a project identifier, not an authorisation | **Do not rotate.** Verify restrictions only |

### Scope of the exposure for `863b98dead568420`

Searched every repo, all branches, full history:

```
SaaradhiGo-mobile    2 commits touched it, 2 files at HEAD before this batch
SaaradhiGo-driver    0
SaaradhiGo-web       0
SaaradhiGo-backend   0
SaaradhiGo-docs      0
```

**Re-scanned 2026-10-17 across every branch, local and remote. It is still
live at the tip of:**

```
main                                origin/main
dev                                 origin/dev
ci/rider-app-pipeline               origin/ci/rider-app-pipeline
fix/brand-consistency-saaradhigo    origin/fix/brand-consistency-saaradhigo
                                    origin/srini_fixes_mobile
```

Two files on each. Only `feat/android-pilot-batch1` is clean, so **anyone
building the rider app from `main` or `dev` today still ships this key** — and
it is in the public history regardless, in any clone, fork, CI cache or APK
ever built from it. Removing it from HEAD does not change that.
**Rotation is the only fix.**

---

## 2. Where it was used — and it was not

Both call sites were dead:

- **Android:** `com.google.android.geo.API_KEY` meta-data feeds the **Maps SDK
  for Android**. The rider app has no `google_maps_flutter` dependency; it
  renders maps with `flutter_map` over OpenStreetMap tiles. Nothing read it.
- **iOS:** `GMSServices.provideAPIKey(...)` in `AppDelegate.swift`, with
  `import GoogleMaps` — and **no GoogleMaps entry in `ios/Podfile` or
  `Podfile.lock`**, so that file could not have compiled. Dead twice over.

**Consequence for rotation: nothing in the rider app breaks when this key is
revoked.** That makes this the cheapest possible rotation — do it first, before
the harder ones.

### What *does* use a Maps key

| Consumer | Key source | Status |
|---|---|---|
| **Backend** — Distance Matrix, geocode, reverse-geocode, place details, directions | `GOOGLE_MAPS_API_KEY` environment variable (`base/settings.py:724`), never in source | **QA has no such variable set.** Verified in the QA service's variable list |
| **Driver app** (`origin/dev`, the recommended canonical app) — **none: it uses `flutter_map` over OSM** | n/a | Needs **no** Maps key |
| **Driver app** (`feat/driver-app-rebuild`) — Maps SDK for Android | `${MAPS_API_KEY}` Gradle manifest placeholder, passed at build time | No key committed; empty by default |
| **Rider app** | — | None needed |

> **Finding worth acting on separately:** because QA has no
> `GOOGLE_MAPS_API_KEY`, `get_google_maps_distance()` returns `(None, None)` on
> every QA request and every QA fare is computed from the **haversine × 1.4
> fallback**, not real road distance. The live QA quote in this batch shows it:
> a client-claimed 2.4 km was server-validated to 3.27 km. Device testing on QA
> therefore does **not** exercise the Distance Matrix path at all. Decide
> whether the pilot's QA should have its own restricted key.

---

## 3. Required restrictions (apply to every key, old and new)

A key with no restrictions is a billable resource anyone who reads your APK can
spend. Set both kinds — application *and* API:

### Android app keys (driver app; rider app only if it adopts the SDK)

- **Application restriction:** *Android apps*. Add one entry per build:
  - package `com.saaradhigo.driver` + the **SHA-1** of the upload/release key
  - package `com.saaradhigo.driver` + the SHA-1 of the **debug** key, for dev
    builds only, and remove it before the pilot
  - (rider, if ever needed) `com.deeptrics.saaradhigo` + its SHA-1
- **API restriction:** *Maps SDK for Android* only. Nothing else.

Get a SHA-1 from a keystore with:
`keytool -list -v -keystore <file> -alias <alias>` — never paste the keystore
or its password anywhere.

### Backend server key (`GOOGLE_MAPS_API_KEY`)

- **Application restriction:** *IP addresses*, limited to the egress IPs of
  whatever runs the backend. **Caveat:** Railway does not give a service a
  stable outbound IP by default, so an IP restriction will break QA unless a
  static egress is configured. Until then, rely on the API restriction below
  plus a quota cap, and treat the key as medium-sensitivity.
- **API restriction:** only what the backend actually calls — *Distance Matrix
  API*, *Geocoding API*, *Places API*, *Directions API*. Nothing else.
- **Quota caps:** set a daily request cap per API. The backend calls Distance
  Matrix on **every fare quote**, including every fallback probe, so an
  uncapped key is the single largest runaway-cost risk in the platform.
- **Billing alert** on the Cloud project.

### iOS keys

Not applicable yet — no iOS build exists. If one is made, restrict by bundle
ID and to *Maps SDK for iOS*.

---

## 4. Rotation procedure for `863b98dead568420`

Low risk, because nothing uses it. Do it in this order so there is never a
window where a new key is unrestricted.

0. **Verify there are no active dependencies first** — do not revoke
   before this. From the repositories and the QA environment:

   | Consumer | Uses a Maps key? | Evidence |
   |---|---|---|
   | Rider app | **No** — `flutter_map`/OSM; both call sites were dead | `pubspec.yaml`; `import GoogleMaps` with no Pod |
   | Driver app (`origin/dev`) | **No** — `flutter_map` | `pubspec.yaml`; no `geo.API_KEY` in its manifest |
   | Driver app (rebuild) | Yes, but build-time and empty by default | `build.gradle.kts` placeholder |
   | Backend | `GOOGLE_MAPS_API_KEY` env only, never in source | `settings.py:724` |
   | QA environment | **Variable not set at all** | QA service variable list |

   So within these five repositories and QA, the compromised key has **no
   active dependency**. That is a strong signal, not a proof — only the Cloud
   console can show whether something outside these repos calls it, which is
   step 1.

1. **Confirm the blast radius in Cloud.** In *APIs & Services → Credentials*,
   find the key with that fingerprint. Check *Metrics* for the last 30 days.
   Expected: no traffic, or only historical traffic. **If it shows recent
   traffic, stop and find out what is calling it before deleting** — something
   outside these five repos is using it.
2. **Create the replacements first, already restricted.** One key per purpose,
   never one shared key:
   - `saaradhigo-driver-android` → Android apps + Maps SDK for Android
   - `saaradhigo-backend-server` → Distance Matrix / Geocoding / Places /
     Directions, with quota caps
   - (optional) `saaradhigo-backend-qa` → same APIs, separate quota, so QA
     spend is visible and capped separately
3. **Deliver them without source control:**
   - backend: set `GOOGLE_MAPS_API_KEY` as a Railway variable on the `backend`
     service (QA) — note this is a variable change, and applying it is a
     deploy, so it waits for the deploy gate
   - driver app: add `MAPS_API_KEY` as a GitHub Actions secret and pass
     `-Pmaps.apiKey=...` at build time (the Gradle placeholder already exists)
4. **Delete the old key.** Not "disable" — delete. A disabled key can be
   re-enabled by anyone with console access who does not know the history.
5. **Verify:** driver app map still renders with the new key; a backend fare
   quote returns `pricing_source: db` with a plausible `validated_km`; old key
   gone from *Credentials*.
6. **Record the rotation date** in this file. Do not record the key.

### What must NOT be done

- **Do not rewrite git history** to remove the key. It does not help — the key
  is already out — and it breaks every clone and open branch. Rotation is the
  fix; history rewriting is theatre.
- **Do not commit the new key**, to any repo, including a `.env` that is
  gitignored today. The gitignore is one mistake away from not covering it.
- **Do not reuse one key across the apps and the backend.** Separate keys are
  what make a future exposure cheap to contain.

---

## 5. What was changed in the repos (batch 1)

| File | Change |
|---|---|
| `SaaradhiGo-mobile/android/app/src/main/AndroidManifest.xml` | Literal key → `${MAPS_API_KEY}` placeholder, supplied empty by Gradle |
| `SaaradhiGo-mobile/android/app/build.gradle.kts` | Supplies the placeholder from `-Pmaps.apiKey`, defaulting to empty |
| `SaaradhiGo-mobile/ios/Runner/AppDelegate.swift` | Dead `GoogleMaps` import and `provideAPIKey` call removed entirely |
| `SaaradhiGo-mobile/test/android_manifest_test.dart` | Asserts no `AIza` literal in the manifest, and that every placeholder is supplied by Gradle |

`google-services.json` is left exactly as it is: that key is meant to ship.

**Note:** these changes are on `feat/android-pilot-batch1` only. `main`,
`dev` and three other branches still carry the key at their tips.

**None of this reduces the exposure of `863b98dead568420`. Only step 4 above
does.**

---

## 6. Fingerprints for the Android restriction (step 2)

Upload-key SHA-1s from the keystores in `android-signing-setup.md`. Not
secrets.

| App | Package | Upload key SHA-1 |
|---|---|---|
| Rider | `com.deeptrics.saaradhigo` | `71:3A:45:6A:B2:C0:C4:FF:FF:43:FF:A6:E5:66:30:37:A2:D9:2B:1D` |
| Driver | `com.saaradhigo.driver` | `94:1B:99:B0:32:51:CF:09:92:A3:AE:C3:C3:B5:8A:FD:60:27:51:0F` |

⚠ **Under Play App Signing the fingerprint that matters at runtime is
Google's app signing key, not the upload key above.** Add the app signing key
SHA-1 from Play Console too, or the map is blank in production while working
in every local build — the most common way Maps restrictions get
misconfigured.

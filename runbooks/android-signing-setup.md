# Android release signing — status and setup

Written for: the owner, plus whoever configures CI.

**No key, password or keystore appears in this document, in any repository, or
in any chat.**

---

## 1. Status: keys generated, outside every repository

Two upload keystores were generated locally with the JDK's `keytool`
(Microsoft OpenJDK 17). They are **not** in any git repo and never will be —
`key.properties`, `*.jks` and `*.keystore` are gitignored in both apps, and a
test asserts no keystore exists in either tree.

```
C:\Users\srini\SaaradhiGo-signing\
    saaradhigo-rider-upload.jks     (4,490 bytes)
    saaradhigo-driver-upload.jks    (4,492 bytes)
    credentials.txt                 <- aliases + passwords, generated randomly
```

| | Rider | Driver |
|---|---|---|
| Alias | `saaradhigo-rider-upload` | `saaradhigo-driver-upload` |
| Algorithm | RSA 4096, SHA384withRSA | RSA 4096, SHA384withRSA |
| Store type | PKCS12 | PKCS12 |
| Valid until | **2054-03-03** | **2054-03-03** |
| SHA-1 | `71:3A:45:6A:B2:C0:C4:FF:FF:43:FF:A6:E5:66:30:37:A2:D9:2B:1D` | `94:1B:99:B0:32:51:CF:09:92:A3:AE:C3:C3:B5:8A:FD:60:27:51:0F` |
| SHA-256 | `D1:F5:01:28:52:3A:63:E4:E0:43:C3:E2:D6:32:B4:92:A2:67:D1:69:BC:7B:F0:55:BD:4C:2E:35:DD:25:30:9D` | `0F:75:1C:F6:9E:74:84:AA:E2:D9:DC:D0:72:EE:55:D6:3A:C0:48:66:CC:6C:A6:31:08:10:69:1F:10:04:4E:B8` |

Fingerprints are **not secrets** — Google publishes them for every app. They
are recorded here because you will need them for Maps key restrictions and for
Play.

Passwords were generated as 32 random characters and written only to
`credentials.txt`. They were never printed to a terminal, never logged, and
are not in this file.

### Your first action: get these off this machine

`C:\Users\srini\SaaradhiGo-signing\` is one disk failure from losing the
ability to update your own apps. Before anything else:

1. Copy both `.jks` files and `credentials.txt` into a password manager or an
   encrypted vault that more than one trusted person can reach.
2. Keep a second offline copy.
3. **Losing an upload key is recoverable** (Play can reset it); losing an *app
   signing* key without Play App Signing is not. See §4.

Nothing was uploaded anywhere — no cloud service, no CI, no chat.

---

## 2. CI secret configuration

Both repos' `build.gradle.kts` already read these four environment variables,
and refuse to produce a release build when they are absent. Nothing in the
code needs to change.

### Per repository (`SaaradhiGo-mobile`, `SaaradhiGo-driver`)

GitHub → Settings → Secrets and variables → Actions → New repository secret:

| Secret | Value |
|---|---|
| `ANDROID_KEYSTORE_BASE64` | `base64 -w0 saaradhigo-<app>-upload.jks` (≈5,990 chars) |
| `ANDROID_KEY_ALIAS` | `saaradhigo-<app>-upload` |
| `ANDROID_KEY_PASSWORD` | from `credentials.txt` |
| `ANDROID_STORE_PASSWORD` | from `credentials.txt` |

Use the **right keystore per repository**. One key per app: a single shared
key means one compromise takes out both listings.

### The workflow step that materialises it

Add to the `release-config` job (or a new `release-build` job) **before** the
build step:

```yaml
      - name: Materialise the upload keystore
        if: ${{ secrets.ANDROID_KEYSTORE_BASE64 != '' }}
        env:
          KEYSTORE_B64: ${{ secrets.ANDROID_KEYSTORE_BASE64 }}
        run: |
          echo "$KEYSTORE_B64" | base64 -d > "$RUNNER_TEMP/upload.jks"
          echo "ANDROID_KEYSTORE_PATH=$RUNNER_TEMP/upload.jks" >> "$GITHUB_ENV"

      - name: Build a signed release APK
        if: ${{ secrets.ANDROID_KEYSTORE_BASE64 != '' }}
        env:
          ANDROID_KEY_ALIAS: ${{ secrets.ANDROID_KEY_ALIAS }}
          ANDROID_KEY_PASSWORD: ${{ secrets.ANDROID_KEY_PASSWORD }}
          ANDROID_STORE_PASSWORD: ${{ secrets.ANDROID_STORE_PASSWORD }}
        run: flutter build apk --release
```

Notes that matter:

- `$RUNNER_TEMP` is wiped at the end of the job and is outside the checkout,
  so the keystore can never be committed by a careless `git add -A`.
- Never `echo` a secret. GitHub masks known secret values in logs, but a
  base64 chunk split across lines can defeat the masker.
- Do **not** add these secrets to a workflow that runs on
  `pull_request_target` or on forks.
- The existing `release-config` job must keep running **without** the secrets:
  it proves an unsigned release build is refused, and that assertion is worth
  more than a second signed build.

---

## 3. Local release builds

Create `android/key.properties` in either app (already gitignored):

```properties
storeFile=C:/Users/srini/SaaradhiGo-signing/saaradhigo-rider-upload.jks
storePassword=<from credentials.txt>
keyPassword=<from credentials.txt>
keyAlias=saaradhigo-rider-upload
```

To build a release APK *without* a keystore — for a smoke test only — use
`flutter build apk --release -PallowDebugSigning=true`. It logs a warning
every time and the artifact must never be distributed.

### Verified on this machine, 2026-10-16

The earlier report that local Android tooling could not produce a release
build is **no longer true**. The missing pieces were SDK components, which
the first attempts downloaded and accepted (NDK r27, Platform 35, CMake
3.22.1). Both apps now build:

| App | Command | Result |
|---|---|---|
| Driver | `flutter build apk --release` | **Refused**, with the signing guard's own message — the guard works against real Gradle, not only in the text test |
| Driver | `flutter build apk --release -PallowDebugSigning=true` | Built, 60.0 MB, 241 s |
| Rider | `flutter build apk --release` | **Refused**, same message, 16 s |
| Rider | `flutter build apk --release -PallowDebugSigning=true` | Built, 61.0 MB, 179 s |
| Rider | `flutter build appbundle` | **Refused** — the guard covers the AAB path too, which is the one Play needs |

`apksigner verify --print-certs` on the opt-out APK reports
`CN=Android Debug`, confirming the warning tells the truth. Both artifacts
were deleted afterwards: a 60 MB file called `app-release.apk` is exactly
what gets sideloaded by mistake.

Both keystores were verified readable with the passwords in
`credentials.txt`, with the expected aliases, valid to 2054:

| App | Alias | SHA-1 |
|---|---|---|
| Rider | `saaradhigo-rider-upload` | `71:3A:45:6A:B2:C0:C4:FF:FF:43:FF:A6:E5:66:30:37:A2:D9:2B:1D` |
| Driver | `saaradhigo-driver-upload` | `94:1B:99:B0:32:51:CF:09:92:A3:AE:C3:C3:B5:8A:FD:60:27:51:0F` |

### Two things to settle before the first upload

- **Play needs an `.aab`, not an `.apk`.** New apps have been App Bundle only
  since August 2021, so the Play command is `flutter build appbundle
  --release`, with the APK kept for sideloaded pilot distribution.
- **60 MB is a universal APK** carrying native libraries for every ABI. For
  drivers on metered data, `--split-per-abi` (or the AAB, which Play splits
  server-side) cuts that to roughly a third. Worth doing before asking anyone
  to download it over mobile data.
- **The driver app's `applicationId` is still the generated
  `com.saaradhigo.saaradhigo_driver`.** An applicationId can never change
  after the first upload. If the pilot should ship as `com.saaradhigo.driver`,
  that has to be decided *before* the first release reaches the Play Console.
  Flagged in `android/app/build.gradle.kts` rather than changed, because it is
  a product decision and the upload key does not care which id it signs.

---

## 4. Play App Signing — what it changes

Google Play now enrols every new app in **Play App Signing**. Two keys exist:

| Key | Who holds it | What it does |
|---|---|---|
| **Upload key** (what was generated here) | you | signs what you upload to Play |
| **App signing key** | Google | re-signs what users install |

Consequences worth understanding before the first upload:

1. **The keystores above are upload keys.** If one is lost or compromised, you
   request a reset in Play Console, register a new upload key, and carry on.
   Before Play App Signing this was fatal; now it is a support ticket.
2. **Users' devices verify Google's signing key, not yours.** So anything that
   depends on a release certificate fingerprint — Google Maps Android key
   restrictions, Firebase, SafetyNet/Play Integrity, Google Sign-In — must use
   the **app signing key's SHA-1 from Play Console**, not the upload
   fingerprints in §1. Getting this wrong is the classic "works in debug,
   blank map in production" bug.
3. **You upload an AAB, not an APK.** `flutter build appbundle --release`.
   APKs remain right for direct-install device testing, which is what the
   pilot checklist uses.
4. **Enrolment is one-way per app.** Decide before the first production
   upload; internal testing tracks are also signed by Play.
5. **The 2054 expiry is for the upload key.** The app signing key's validity is
   Google's concern once enrolled.

**Recommended:** accept Play App Signing. For a pilot with one developer
machine, Google holding the irreplaceable key is a reduction in risk, not an
increase.

---

## 5. What still needs you

| # | Action | Why only you |
|---|---|---|
| 1 | **Back up both keystores and `credentials.txt`** off this machine | They exist in exactly one place right now |
| 2 | Add the four secrets per repository | Repository admin |
| 3 | Create the Play Console account and both app listings | Commercial identity + payment |
| 4 | After the first upload, copy the **app signing key SHA-1** from Play Console into the Maps key restrictions | Only visible in Play Console |
| 5 | Decide whether a second person can reach the vault | Bus factor |

**Not done, deliberately:** nothing was uploaded to any external service, no
Play Console account was touched, and no production signing key was created —
these are upload keys, which is what a pilot needs.

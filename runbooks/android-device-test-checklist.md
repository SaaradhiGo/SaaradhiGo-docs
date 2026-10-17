# Android device acceptance checklist

Written for: whoever has a physical Android phone and a QA build.

Everything here is **UNKNOWN** and cannot be cleared from the development
machine: `flutter build apk` fails there ("Building with plugins requires
symlink support. Please enable Developer Mode"), there is no emulator or
device attached, and there is no macOS. CI builds a **debug** APK only.

Mark each line PASS or FAIL with device, OS version and build. Do not mark
anything PASS by reasoning — every item exists because it cannot be proven any
other way.

---

## 0. Before you start

- [ ] Note phone model, Android version, and whether it is an OEM build with
      aggressive battery management (Xiaomi / Vivo / Oppo / Realme / Samsung).
      **At least one Android 14+ (API 34) device is required** — the
      foreground-service fix only matters from there.
- [ ] Build the QA artifacts with the build script in each app, not by hand:

      ```
      tool/build_release.sh qa-apk      # signed, needs the backup gate cleared
      tool/build_release.sh smoke-apk   # debug-signed, for a pipeline check
      ```

      Both produce `--split-per-abi` APKs. Install the **arm64-v8a** one on a
      modern phone (22 MB for the driver, 24 MB for the rider, against ~60 MB
      for a universal build). `qa-apk` refuses to run until
      `SAARADHIGO_KEYSTORE_BACKUP_VERIFIED=yes` is set — see
      `android-signing-setup.md`.
- [ ] Confirm the build says it is QA **on the device, not from the command
      you typed**: the driver app shows a red `QA` corner banner on every
      screen, and the rider app shows its environment badge. A QA build with
      no banner means the artifact is not what you think it is.
- [ ] Have QA accounts ready — `qa-test-accounts.md`. Route A (phone + OTP
      bypass) works today; Route B (email) needs email delivery configured.

> ### Two QA limitations that will shape what you can test
>
> **1. QA has no `GOOGLE_MAPS_API_KEY`, and this is worse than a fare
> fallback.** Verified from the Railway service's variable list, not assumed.
> All four map proxies return **HTTP 502 `API_ERROR`** when the key is unset:
>
> | Route | Rider-visible effect on QA today |
> |---|---|
> | `/ride/maps/geocode` | destination search returns nothing |
> | `/ride/maps/place-details` | a chosen suggestion cannot be resolved |
> | `/ride/maps/reverse-geocode` | the pickup pin shows no address |
> | `/ride/maps/directions` | no route line, no ETA |
>
> The **rider** app calls all four. The **driver** app calls none of them, so
> driver-side testing is unaffected. Fares also fall back to haversine × 1.4,
> so `validated_km` will differ from real road distance — that part is the
> fallback working.
>
> **A rider cannot book by typing a destination on QA until a restricted QA
> Maps key exists.** Decide before the device pass whether to create one (see
> `maps-key-rotation.md`) or to certify booking by map pin only and record the
> search path as UNTESTED.
>
> **2. No email will arrive until SES is configured.** Run
> `python manage.py email_safety_check --to <the QA address>` against QA
> first: it reports whether the deployment can send and whether that address
> is allowed, without sending anything.

---

## 1. Driver onboarding and document upload

- [ ] Fresh driver account; OTP login succeeds. *(First-time check — driver
      login read the wrong token keys until this batch, so it had never
      worked.)*
- [ ] Session survives past 15 minutes without a re-login — proves token
      refresh works. *(Also a first-time check, same cause.)*
- [ ] Profile shows "Documents needed" and an **Upload my documents** button.
- [ ] Camera capture works for the licence front: system camera opens, photo
      returns, tile shows progress then **Uploaded**.
- [ ] Gallery picking works for the licence back.
- [ ] Tile says **Uploaded**, never "Verified" or "Approved".
- [ ] Submit is disabled while any upload is in flight.
- [ ] **Airplane mode mid-upload:** that document shows a named failure, the
      others keep their state, and retry works.
- [ ] A >10 MB photo is refused locally, before any network call.
- [ ] A HEIC file is refused with "That photo format is not supported", not an
      S3 signature error.
- [ ] Documents appear in the Django console's KYC queue and are viewable.

## 2. Vehicle registration and admin approval

- [ ] Submitting the licence leads to **Add my vehicle**.
- [ ] Registration number upper-cases as typed; a 4-character number is
      refused with "That looks too short".
- [ ] All five Indian vehicle documents are offered (RC, insurance, permit,
      fitness, PUC).
- [ ] Vehicle submits; the confirmation names the review step.
- [ ] Console: the vehicle and its documents appear, and the compliance
      checklist reflects exactly the five India documents — no PUC/permit/
      fitness row for a US driver, no "inspection" row for an Indian one.
- [ ] Console: approve the driver. The app's profile flips to **Approved**
      after a refresh.
- [ ] A driver with a missing document cannot be approved, and the console
      names which one.

## 3. Ride booking and dispatch

- [ ] Rider books; the driver receives the offer with the app in the
      **foreground**.
- [ ] The offer shows fare **with a currency symbol** and distance **with a
      unit**.
- [ ] Driver accepts; the rider sees driver and vehicle details.
- [ ] Driver declines; the offer passes to another driver (needs two drivers).
- [ ] Nobody accepts: the request times out and the rider is told.
- [ ] Rider cancels while searching — the offer is withdrawn from the driver's
      screen.
- [ ] Driver cancels after accepting — the rider is told and re-dispatched.
- [ ] Two riders booking simultaneously are not matched to the same driver.

## 4. Background GPS on Android 14+

- [ ] Go on duty. The app does **not** crash.
      *(Before this batch: `MissingForegroundServiceTypeException` on API 34+.)*
- [ ] A persistent notification appears while on duty.
- [ ] Lock the screen for 10 minutes; location still reaches the backend
      (console live map or GPS trail).
- [ ] Background the app and open 3–4 other apps to force memory pressure;
      duty survives.
- [ ] **Reboot the phone while on duty** and record what happens — the
      manifest requests `RECEIVE_BOOT_COMPLETED` and the intended behaviour is
      unverified.
- [ ] On an OEM build with battery optimisation **not** exempted, record how
      long duty survives. This number decides whether the rationale screen is
      enough.
- [ ] GPS reconnects after a tunnel / signal loss without the driver acting.

## 5. Push notifications

Standing rule: **do not mark a background push PASS without observing it on a
device.**

- [ ] Ride offer arrives with the driver app **backgrounded**.
- [ ] Ride offer arrives with the driver app **killed** (swiped away).
- [ ] Rider receives trip-status pushes (accepted, arrived, started,
      completed).
- [ ] Notification permission prompt appears on first run (Android 13+), and
      the app behaves sensibly if it is denied.

## 6. Rider SOS and operational alert receipt

- [ ] During a ride, tap SOS. The sheet opens with **Call 112** first.
- [ ] Raise a categorised SOS. A reference number appears (`#<id>`), not a
      bare reassurance.
- [ ] **Console / SOS admin: the event exists**, with the trip and the
      coordinates attached, and the category the rider chose.
- [ ] Ops receive the alert (push and/or the ops topic) — confirm with the
      person on duty, not by inference.
- [ ] **Airplane mode, then tap SOS:** the sheet shows a failure AND a
      "Call 112 now" button. It must never say the alert was sent.
      *(This is the exact regression: the old screen always claimed success.)*
- [ ] **Call 112** opens the dialler with 112 pre-filled. **Do not place the
      call.**

## 7. Emergency-contact SMS delivery

- [ ] The rider's profile has an emergency contact saved.
- [ ] After an SOS, that contact **receives the SMS**. Confirm on the
      recipient handset.
- [ ] The message names the rider and the SOS reference and does not contain a
      raw internal id or a URL that 404s.
- [ ] With no emergency contact set, the SOS still succeeds and ops are still
      alerted.

> This path has never been exercised from the rider side — the rider app had
> no SOS call until this batch. Treat a failure here as expected-until-proven.

## 8. Fare, payment, completion and receipt

- [ ] Quote shows **₹**, not the letters `INR`.
      *(The regression: the backend sends an ISO code and the app rendered it
      as a symbol — `INR169.02`.)*
- [ ] Itemised breakdown lines all show ₹.
- [ ] base + distance + time + adjustments **equals** the quoted total,
      exactly. Deployed QA returns 60.00 + 55.59 + 19.62 + 33.80 surge + 0.01
      rounding = 169.02.
- [ ] Trip start requires the rider's OTP, and a wrong one is refused.
- [ ] Trip completes; the driver sees a summary and earnings update.
- [ ] Cash collection is confirmed by the driver.
- [ ] Rider receives the emailed receipt. **Blocked until email delivery is
      configured** — see §9.
- [ ] Receipt shows ₹ and GST, and its total matches what was charged.
- [ ] Online payment: expect a refusal, since Cashfree is sandbox-only and
      live payments are held. A `CURRENCY_NOT_SUPPORTED` refusal for a non-INR
      trip is correct behaviour, not a bug.

## 9. Authentication (new in this batch)

- [ ] **Rider, Mobile tab:** OTP login still works. This is the primary route
      — a regression here is worse than any new feature.
- [ ] **Driver, Mobile tab:** OTP login works *(first-time check — see §1)*.
- [ ] **Rider, Email tab:** login with a QA email account succeeds.
- [ ] **Driver, Email tab:** same.
- [ ] A wrong password shows **one** generic message, identical to the message
      for an unknown address.
- [ ] "Forgot password?" shows the same confirmation for a known and an
      unknown address.
- [ ] A reset link sets the password, works **once**, and the old password no
      longer works. **Needs email delivery, or the shell workaround in
      `qa-test-accounts.md`.**
- [ ] Switching tabs does not carry a validation error across.
- [ ] **Operator console:** password + TOTP still required; neither alone
      works.
- [ ] **Operator email login:** refused with `AUTH_MFA_REQUIRED` when
      `mfa_code` is absent; succeeds with a valid code.
- [ ] A support-level operator cannot create an operator account (403).
- [ ] An invited account's response contains **no password**.

### Ending a session — the part only a device can prove

Both apps were changed so that a secure-storage failure cannot bring a
logged-out session back. Tests simulate a store that throws; a real Android
Keystore cannot be made to throw on demand, so these are the device checks
that matter.

- [ ] **Log out, force-stop, reopen.** Both apps land on the login screen,
      not on a restored session.
- [ ] **Log out while offline** (aeroplane mode). Logout still completes and
      the app lands on the login screen — the server call is best-effort and
      must not block it.
- [ ] After a logout, the refresh token is **rejected server-side**: the old
      session cannot be resumed from another client. (Ops can confirm the
      token is blacklisted; the app calls `POST /auth/logout/` now.)
- [ ] **Reinstall over the top** (`adb install -r`, same signing key). A
      session from before the upgrade behaves as expected and no stale
      credential is silently reused.
- [ ] A wrong **email** password does *not* produce "Your session has
      expired." It must say the credentials were wrong. (The refresh path used
      to run on a rejected email login.)
- [ ] With the app signed in, change the device screen lock / re-enrol
      biometrics, then reopen. This is the real-world trigger for a Keystore
      entry becoming undecryptable: the app must either keep working or end
      the session cleanly, never restore a half-session.

## 10. Release build and store artifact

- [ ] CI `release-config` job green on the branch — it proves the signing
      guard refuses an unsigned release build **with no secrets configured**,
      and that the documented opt-out still works.
- [ ] `tool/build_release.sh qa-apk` produced the installed artifact.
- [ ] A signed release APK installs on a clean device, **and** installs over
      an earlier build of the same key (`adb install -r`) without a signature
      error. An upgrade failure here is how a pilot ends with drivers unable
      to take an update.
- [ ] `apksigner verify --print-certs` shows **your** key, not
      "CN=Android Debug". The build script prints this for every split APK;
      read it rather than trusting the command you typed.
- [ ] A release build with the production URLs but **without**
      `ALLOW_NON_PRODUCTION_HOST` refuses to start when pointed at QA — and a
      production-pointed build starts normally.
- [ ] A release build compiled with **no** `--dart-define` at all shows the
      "This build cannot be used" screen rather than silently using the dev
      API.

### Before the first Play upload

- [ ] **`.aab`, not `.apk`.** New apps have been App Bundle only since August
      2021. `tool/build_release.sh play-aab` is the command; the APK is for
      sideloaded pilot distribution only.
- [ ] **The driver `applicationId` is settled.** It is still the generated
      `com.saaradhigo.saaradhigo_driver` and can never be changed after the
      first upload. See the recommendation in
      `ci-and-release-prep-batch4-report.md`.
- [ ] **Both keystores are backed up** off this machine, with
      `credentials.txt`, somewhere a second person can reach. The build script
      refuses to produce a signed artifact until
      `SAARADHIGO_KEYSTORE_BACKUP_VERIFIED=yes` says this is done.

---

## Still UNKNOWN after this checklist

- **iOS, entirely.** The driver app's `Info.plist` is the stock template — no
  `NSLocation*UsageDescription`, no `UIBackgroundModes` — so it cannot request
  location. Neither app has a `DEVELOPMENT_TEAM`. The rider app's dead
  GoogleMaps import was removed this batch, but iOS remains unbuilt and
  unbuildable here.
- **Sustained multi-driver dispatch on real devices.** The isolated load stack
  exercises the backend with synthetic clients and excludes auth entirely.
- **Any payment beyond cash.** Cashfree is sandbox-only; live payments held.
- **Email delivery.** Nothing will arrive until SES/SMTP is configured.

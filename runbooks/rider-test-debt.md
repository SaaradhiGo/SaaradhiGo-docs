# Rider app test debt

Status as of 2026-09-24. Rider app: `SaaradhiGo-mobile`, branch `dev`.

## Why this document exists

Nineteen tests in the rider app were quarantined. Every quarantine carried a
stated reason, and **the stated reasons were wrong**. They were written when no
Flutter toolchain was available on this machine, so nobody had ever seen the
tests fail. The reasons were inferences, and they sent the reader in the wrong
direction:

- Seven ride-state-recovery tests were blamed on "shared-preferences mock state
  appears to leak between tests". No leak existed. The app was wrong.
- Twelve screen tests were blamed on asserting "against a redesign whose widget
  tree differs from what ships" and "needs a device to see the real layout".
  They are widget tests. They never needed a device. The layouts they assert
  against are the ones that ship.

A quarantine reason that is a guess is worse than no reason, because it stops the
next person from looking. This file records what is actually true.

## Toolchain

The precondition for all of this was a working Flutter install, which now exists:

```
Flutter 3.47.5 - channel stable - revision 6a19cca564
Tools - Dart 3.13.4 - DevTools 2.60.0
Installed at C:\src\flutter (git clone --depth 1 -b stable)
```

Satisfies both apps: rider needs Dart `^3.11.1`, driver needs `^3.12.2`.

`flutter test` and `flutter analyze` work. **`flutter build apk` does not** — see
[Blocker: Windows symlink support](#blocker-windows-symlink-support).

## Where the numbers stand

| | Before | After |
|---|---|---|
| Passing | 31 | **50** |
| Skipped test bodies | 19 | **8** |
| Suites skipped wholesale | 3 | 0 |

`flutter analyze`: no errors, no warnings, 29 pre-existing `info` lints.

## Classification

Categories, defined here because the quarantine had none:

- **A — App defect the test correctly caught.** The test was right.
- **B — Test asserted something impossible.** Self-contradictory or wrong fixture.
- **C — Test harness incomplete.** Missing providers, wrong surface, bad stub.
- **D — Test hook missing from the app.** Assertion needs a key that was never added.
- **E — Test asserts a feature that does not exist.** Product question, not a bug.
- **F — Flaky.** Passes or fails on environment, not on behaviour.
- **G — Still unexplained.** Honest unknown.

### Ride state recovery — 7 tests, now all passing

`test/ride_state_recovery_test.dart`. Quarantine reason (mock leak) was false.
These were **category A**, and they were guarding four real rider-visible
defects:

| # | Defect | Rider consequence |
|---|---|---|
| A1 | Both recovery paths dropped the trip id. `syncStateFromBackend` receives it as a parameter, `syncStateFromActiveTripResponse` parses it into a local, neither stored it in state. `_saveActiveTripId()` only keeps `active_trip_id` while `state.tripId` is set, so recovering a ride **deleted the pointer it had just recovered from**. `LifecycleObserver` then had nothing to re-persist on background. | App killed mid-ride → ride lost on the next offline start. |
| A2 | Status vocabularies did not match. Backend `TripStatus` choices are `requested`/`accepted`/`reached`/`in_progress`/`completed`/`cancelled`. Both recovery ladders branched on `'arrived'` and `'started'`, which the backend never sends, and fell through to the current status otherwise. On a cold start that is `RideStatus.none`. `'requested'` was unmapped too. The live WebSocket handler had `'reached'` right; only recovery was wrong. | **App restarts while the driver waits at the pickup point → rider sees no active ride at all.** Recovering mid-search failed the same way. |
| A3 | `clearState()` was `void` while firing two un-awaited SharedPreferences writes. Callers could not know when the pointer was gone, and the writes raced. | Cancel or completion followed closely by an app kill could leave a finished trip's id on disk. |
| A4 | The cancelled-overlay auto-dismiss used bare `Future.delayed`: uncancellable, and it wrote to a disposed notifier if the rider left within three seconds. | Thrown exception on a disposed notifier; two cancellations left two timers racing. |

Also fixed alongside: `RideState` was `@immutable` with no `operator ==`, so it
compared by identity and `NotifierProvider` rebuilt every listener on every
write — once per driver-location tick on the screen a rider watches for an entire
ride.

Test-side corrections (**category B**), which are corrections to the tests, not
to production behaviour:

- `'handles cancelled trip'` asserted `const RideState()` *and*
  `showCancelledOverlay == true` on adjacent lines. Both cannot hold.
- `'saves active trip ID when backend has active trip'` **asserted nothing**. It
  built a fixture, noted in a comment that `RideService` could not be injected,
  and ended — reporting green while covering nothing. `rideServiceProvider`
  solved the injection, so it now asserts.
- Fixtures using `'started'`, `'ride_started'` and `'driver_accepted'` corrected.
  `'ride_started'` is a **WebSocket event type** in `consumers.py`, not a trip
  status; the fixture conflated the two, and the fall-through it hit is what hid
  A2.

Added: 8 regression tests pinning the status vocabulary, one of which fails if the
backend gains a `status_code` the rider app cannot read.

### Screen tests — 12 tests, 5 now passing, 7 still skipped

`test/profile_tab_test.dart`, `test/personal_info_screen_test.dart`,
`test/privacy_security_screen_test.dart`.

The dominant root cause was **category C**: `ProviderNotFoundException`.
`HomeScreen.initState` reads `MapProvider` and `NotificationProvider` in a
post-frame callback, and each suite registered only `AuthProvider`. The tree threw
before rendering, and every "Found 0 widgets with text ..." failure was the
consequence, not the cause.

Fixed by `test/support/screen_test_harness.dart`, which supplies:

- all three providers (`wrapWithRiderProviders`)
- a `FakeNotificationService` so the first frame's fetch never touches the network
- geolocator channel stubs (`stubStartupPluginChannels`)
- a phone-shaped viewport (`usePhoneSurface`, 412x915)

Two harness traps worth knowing, both of which cost real time:

1. **Do not stub `plugins.flutter.io/shared_preferences`.**
   `SharedPreferences.setMockInitialValues()` owns that channel. Stubbing it
   returned an empty store and silently discarded every suite's seeded values.
2. **Use `tester.binding.setSurfaceSize`, not `tester.view.physicalSize`.**
   Setting `physicalSize` alone left layout and hit-testing in different
   coordinate spaces, so `tap()` computed a centre inside the viewport and still
   missed, with "would not hit test on the specified widget".

App-side fixes found along the way:

- **Category A.** `privacy_security_screen.dart` passed each widget's own `key`
  down onto a descendant (`key: key`) at three sites, so the same key appeared
  **twice in the tree** and every `find.byKey()` on it was ambiguous — unusable
  for tests, integration drivers and accessibility tooling.
- **Category D.** `edit_personal_information_screen.dart` had no
  `personal-save` or `personal-phone-display` keys. Added.
- **Category D.** `HomeScreen`'s bottom nav items had no keys, and `_NavItem` did
  not even accept one. Added `home-nav-home/history/wallet/profile`, matching the
  `privacy-nav-*` convention the privacy screen already used.

**Product observation, not a test problem:** the "location permissions are
permanently denied" SnackBar is rendered over the bottom navigation bar and
swallows taps on it for its full duration. On a real handset a rider who denies
location cannot switch tabs until it clears. Denying location is common in India.
Worth a design decision.

#### The 7 still skipped

Each now carries an accurate reason in the source.

| Test | Cat | What actually happens | Next action |
|---|---|---|---|
| profile tab: `renders provider-driven profile values` | G | Tap on `home-nav-profile` hit-tests correctly now, but the tab still does not change, so the profile pane never renders. Reaching the same pane via `/home?tab=3` works and is asserted by a passing test, so this is specifically the tap-to-switch path. | Instrument `onItemTap`/`setState` under test. |
| profile tab: `renders fallback values when profile data is unavailable` | G | Same tap-to-switch cause. | As above. |
| profile tab: `keeps Personal Information menu navigation working` | G | Same tap-to-switch cause. | As above. |
| personal info: `renders fallback values and keeps phone read-only` | B | `'John Doe'` matches twice — the screen shows the name in an editable `TextField` *and* a display `Text`, so `find.text` also matches the `EditableText`. | Scope the finder or key the display Text. |
| personal info: `save triggers loading, calls API, and routes to profile tab` | A? | Reads back `'9876543210'` from SharedPreferences after save and gets `null`. | **Answer this one.** Either the save path does not persist the phone number or the test asserts the wrong key. Worth knowing before trusting profile saves. |
| personal info: `save failure shows error and stays on the same screen` | F | Passes or fails depending on surface size — layout/timing-sensitive around the error SnackBar. | Pin the assertion to the SnackBar. |
| privacy: `toggle updates persist across provider rebuild` | A? | Expects the toggle to read `false` after a provider rebuild and reads `true`. | **Answer this one.** Whether a privacy toggle survives a rebuild is a question about the screen, not the test, and should be settled before these settings are described to a user as saved. |

Plus one **category E**, skipped and correctly so:

| Test | What it assumes |
|---|---|
| personal info: `bottom nav routes to /home?tab=0..3` | That `EditPersonalInformationScreen` has a bottom navigation bar. It does not — its `bottomNavigationBar` slot holds the Save Changes button. The privacy screen *does* have one. Either this screen is missing it or the design changed. A product question; a test should not invent the answer. |

Two of these (`save triggers loading`, `toggle updates persist`) may be real
defects in the profile and privacy screens. They are marked `A?` rather than `A`
because I did not confirm them. Do not read the passing count as evidence that
profile saves and privacy toggles work.

## Blocker: Windows symlink support

`flutter build apk` and `flutter build appbundle` cannot run on this machine.

```
Building with plugins requires symlink support.
Please enable Developer Mode in your system settings.
```

Reproducible, and it stops at the plugin-symlink step *after* dependency
resolution succeeds — so `flutter test` and `flutter analyze` still work, and
`.dart_tool/package_config.json` is written.

Remediation, in order of preference:

1. Enable Windows Developer Mode: `start ms-settings:developers`. Interactive and
   machine-wide.
2. Equivalent registry value, **requires elevation** (this session was not
   elevated, and a machine-wide Windows security setting should not be changed
   silently anyway):
   `HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\AppModelUnlock`
   → `AllowDevelopmentWithoutDevLicense` = `1` (DWORD)
3. Build in CI, where the runner is Linux and the restriction does not apply.

Until one of these is done, **no APK has been produced or installed from this
machine**, and nothing here constitutes on-device verification.

### Side effect to know about

Running `flutter pub get` regenerates the desktop plugin registrants under
`linux/`, `macos/` and `windows/`. Those diffs are line-ending churn only and are
deliberately **not committed**. Reverting them with `git checkout` makes the next
`flutter` invocation re-resolve and hit the symlink error again — let them stay
dirty in the working tree.

## What this does not tell you

- No test here ran on a physical device or emulator.
- 8 test bodies remain unverified, two of which may be real defects.
- `flutter analyze` still reports 29 `info` lints, untouched.
- The 3 screen suites cover profile and privacy settings. They say nothing about
  booking, fare display, the active-ride screen, SOS or payment.

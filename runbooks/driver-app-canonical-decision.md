# Which driver app is canonical — evidence and recommendation

Written for: the owner, who makes this call. **Nothing was merged, rebased or
discarded. Both histories are intact.**

---

## 0. Recommendation

**Make `origin/dev`'s driver app canonical. Port five things to it from
`feat/driver-app-rebuild`, then archive the rebuild branch.**

This reverses the implicit assumption of the last two batches, and it reverses
a claim in my own batch-2 report. The evidence is below, including the two
places that report was wrong.

---

## 1. Two corrections to what I previously told you

| I said | Actually |
|---|---|
| "`origin/dev`: **no tests**" | **290 tests, all passing.** `flutter test` on a clean worktree: `00:19 +290: All tests passed!` |
| "a driver cannot be onboarded — no UI anywhere" (go-live audit, P0 #1) | True of the rebuild branch only. `origin/dev` has a complete presigned-upload stack, a document vault and four onboarding screens |

I had checked `test/` on the rebuild and the *feature* directories on `dev`,
and reported the absence of what I had not looked for. The first correction is
the one that changes the decision.

---

## 2. Side-by-side

| Dimension | `origin/dev` | `feat/driver-app-rebuild` (+ batches 1–2) |
|---|---|---|
| Dart files (excl. `lib_backup`) | **363** | 86 |
| Screens | **35** | 16 |
| Tests | **290 passing** | 153 passing |
| `flutter analyze` | 1 warning (missing `.env` asset) | **clean** |
| CI | **none** | analyze + test + debug APK + release-config |
| **Auth token contract** | **correct** — reads `token` / `refresh_token` | was `access` / `refresh`: **login and refresh could never work**; fixed in batch 2 |
| Email + password login | no | **yes** (batch 2) |
| KYC document upload | **yes** — `core/uploads/presigned_upload_service.dart`, `features/documents`, `features/onboarding` (licence, photo bank, vehicle, verification status) | built in batch 1 |
| Vehicle management | **yes** — `features/vehicle` incl. add-vehicle screen + tests | built in batch 1 |
| Incentives / elite status | **yes** (`features/incentives`, backend routes exist) | no |
| Notifications feature | **yes** | no |
| Trip history + date filters | **yes** (`features/trip_history`) | basic history |
| Support / chat | **yes** (`features/support`, 20 files) | basic |
| Trip OTP + cash confirmation | **yes** | yes |
| GPS / background location | `location_notifier`, throttling, disclosure sheet, + tests | `location_service`, duty controller |
| Dispatch | `incoming_ride_bottom_sheet`, `driver_location_consumer`, WS service + tests | `dispatch_controller`, `ride_offer_sheet` |
| Android 14 foreground service | **already correct** | fixed in batch 1 |
| Release signing | **debug key** | **real signing + guard** (batch 1) |
| Committed API key | none | none |
| Maps stack | `flutter_map` (OSM) — **needs no Google Maps key** | `google_maps_flutter` — needs a key |

### The decisive points

1. **`dev` works against the real API; the rebuild did not.** `dev` reads
   `token`/`refresh_token` in both login and refresh — matching the server and
   matching the live QA probe. The rebuild read `access`/`refresh`, so **no
   driver could ever sign in**, and its own tests hid it by using the same
   wrong keys.
2. **`dev` already has the capability the audit called the #1 blocker.** The
   presigned-upload service even documents the same three-step contract I
   re-derived for the rebuild.
3. **4× the product.** Incentives, notifications, trip history with filters, a
   support surface and 35 screens against 16. Rebuilding that is months, and
   "prefer retaining proven functionality over rebuilding it" points one way.
4. **Test coverage is 2× and passing.** Quality is not the argument for the
   rebuild that it appeared to be.

### What the rebuild is genuinely better at

Engineering hygiene, not product: CI, release signing, a manifest regression
test, and the API-contract discipline. **All five are portable in a day.**

---

## 3. What must be ported to `dev` before it can ship

Nothing here has been applied — `origin/dev` is untouched.

| # | Port | From | Why it matters |
|---|---|---|---|
| 1 | **CI workflow** — analyze, test, debug APK, `release-config` | `.github/workflows/flutter.yml` | `dev` has no CI at all, so its 290 tests run only when someone remembers |
| 2 | **Release signing + guard** | `android/app/build.gradle.kts` | `dev` still hardcodes `signingConfigs.getByName("debug")` — unshippable, and silently so |
| 3 | **`release_signing_test.dart`** | `test/` | Keeps the guard from being deleted later |
| 4 | **Android manifest test** | `test/android_manifest_test.dart` | `dev`'s manifest is correct today; nothing stops a regression |
| 5 | **Email + password login tabs** | `features/auth/` + `api_endpoints.dart` | Required by the auth workstream; `dev` has OTP only |
| 6 | **Auth contract conformance test** | `test/auth_contract_conformance_test.dart` + `test/contracts/` | The thing that would have caught the rebuild's defect in minutes |

Two `dev`-specific items to fix while porting:

- **`.env` as a bundled Flutter asset** (`pubspec.yaml: - .env`, read by
  `feature_flags.dart`). The rider app removed exactly this pattern because
  anything bundled as an asset can be extracted from the APK. Today it carries
  feature flags, not secrets — keep it that way, or move to `--dart-define`.
- **`/driver/incentives/` + `/driver/driver/incentives/` fallback pair.** The
  real route has the doubled segment, so every incentives call makes one
  guaranteed-404 request first. Harmless, wasteful, and worth deleting the
  wrong one.
- **`lib_backup/` (41 files of archived legacy code)** is still in the tree.
  Delete it in its own commit so the diff is legible.

---

## 4. What would be lost by archiving the rebuild

Only the six portable items above — all of which the table says to port. No
product capability in the rebuild is absent from `dev`. I checked each feature
directory individually rather than inferring from the file counts.

The rebuild's history stays on `feat/driver-app-rebuild` and
`feat/android-pilot-batch1`; nothing is deleted.

---

## 5. How this was established

```
git merge-base HEAD origin/dev            -> exit 1 (no common ancestor)
git ls-tree -r origin/dev | grep -c .dart -> 363
git ls-tree -r HEAD       | grep -c .dart -> 86
worktree of origin/dev: flutter test      -> 290 passed
                        flutter analyze   -> 1 warning (.env asset)
endpoint extraction, both apps, compared against servers/*/urls.py
token-key grep across lib/core/network/ and lib/features/auth/ on both sides
```

A throwaway worktree was used for `origin/dev` so nothing in the working repo
changed. It has been removed.

---

## 6. Recommended sequence — awaiting your approval

1. You confirm `origin/dev` as canonical.
2. New branch **from `origin/dev`**: `feat/driver-dev-hardening`.
3. Port items 1–6 above, plus the three `dev`-specific fixes.
4. Run `flutter analyze` + the full 290-test suite, plus the new contract
   conformance test, on that branch.
5. Archive `feat/driver-app-rebuild` with a tag and a note pointing here.
6. Only then does the driver app enter the device-testing gate.

**Stopping here. No branch has been created and `origin/dev` has not been
touched.**
